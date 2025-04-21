---
layout: post
title: "Implementing Hexagonal Architecture with a Library Book Reservation System"
date: 2025-04-21
categories: architecture spring-boot hexagonal
---

# Implementing Hexagonal Architecture with a Library Book Reservation System

In this post, we'll explore how to implement a Hexagonal Architecture (also known as Ports and Adapters) using a practical example of a library book reservation system. We'll use Spring Boot as our framework of choice.

## System Overview

Our library book reservation system has the following requirements:

1. **Primary Ports (Driving Adapters)**:
   - REST API endpoints for:
     - Making book reservations
     - Querying existing reservations
   - AMQP message handler for book availability notifications

2. **Secondary Ports (Driven Adapters)**:
   - JPA for data persistence
   - AMQP for publishing domain events

## Domain Model

Let's start with our core domain entities:

```java
public class Book {
    private UUID id;
    private String title;
    private String isbn;
    private BookStatus status;
    
    // Getters, setters, and domain logic
}

public class Reservation {
    private UUID id;
    private UUID bookId;
    private UUID userId;
    private LocalDateTime reservationDate;
    private ReservationStatus status;
    
    // Getters, setters, and domain logic
}

public enum BookStatus {
    AVAILABLE, RESERVED, CHECKED_OUT
}

public enum ReservationStatus {
    PENDING, COMPLETED, CANCELLED
}

public interface DomainEvent {
    // Marker interface for domain events
}

public interface DomainEventPublisher {
    void publish(DomainEvent event);
}

public class BookAvailableEvent implements DomainEvent {
    private final UUID bookId;
    
    public BookAvailableEvent(UUID bookId) {
        this.bookId = bookId;
    }
    
    public UUID getBookId() {
        return bookId;
    }
}
```

## Core Domain Logic

The heart of our application is the domain service that handles the business logic:

```java
/**
 * Core domain service that handles book reservation business logic.
 * Note: This class is intentionally free of framework annotations (@Component, @Transactional, etc.)
 * to maintain framework independence and adhere to Hexagonal Architecture principles.
 * Framework-specific concerns like dependency injection and transaction management
 * are handled at the infrastructure layer.
 */
public class BookReservationService {
    private final BookRepository bookRepository;
    private final ReservationRepository reservationRepository;
    private final DomainEventPublisher eventPublisher;
    
    public BookReservationService(
        BookRepository bookRepository,
        ReservationRepository reservationRepository,
        DomainEventPublisher eventPublisher) {
        this.bookRepository = bookRepository;
        this.reservationRepository = reservationRepository;
        this.eventPublisher = eventPublisher;
    }
    
    public Reservation makeReservation(UUID bookId, UUID userId) {
        Book book = bookRepository.findById(bookId)
            .orElseThrow(() -> new BookNotFoundException(bookId));
            
        if (book.getStatus() != BookStatus.AVAILABLE) {
            throw new BookNotAvailableException(bookId);
        }
        
        Reservation reservation = new Reservation(bookId, userId);
        reservationRepository.save(reservation);
        
        book.setStatus(BookStatus.RESERVED);
        
        return reservation;
    }
    
    public void handleBookAvailable(UUID bookId) {
        Book book = bookRepository.findById(bookId)
            .orElseThrow(() -> new BookNotFoundException(bookId));
            
        book.setStatus(BookStatus.AVAILABLE);
        
        eventPublisher.publish(new BookAvailableEvent(bookId));
    }
}
```

## Ports and Adapters Implementation

### Primary Ports (Driving Adapters)

1. **REST Controller**:

```java
@RestController
@RequestMapping("/api/reservations")
public class ReservationController {
    private final BookReservationService reservationService;
    
    @PostMapping
    public ResponseEntity<Reservation> makeReservation(
        @RequestBody ReservationRequest request) {
        Reservation reservation = reservationService.makeReservation(
            request.getBookId(), 
            request.getUserId()
        );
        return ResponseEntity.ok(reservation);
    }
    
    @GetMapping("/{userId}")
    public ResponseEntity<List<Reservation>> getUserReservations(
        @PathVariable UUID userId) {
        return ResponseEntity.ok(
            reservationService.getUserReservations(userId)
        );
    }
}
```

2. **AMQP Message Handler**:

```java
@Component
public class BookAvailabilityHandler {
    private final BookReservationService reservationService;
    
    @RabbitListener(queues = "book-availability")
    public void handleBookAvailable(BookAvailableMessage message) {
        reservationService.handleBookAvailable(message.getBookId());
    }
}
```

### Secondary Ports (Driven Adapters)

1. **JPA Repository**:

```java
@Repository
public interface BookRepository extends JpaRepository<Book, UUID> {
    // Custom queries if needed
}

@Repository
public interface ReservationRepository extends JpaRepository<Reservation, UUID> {
    List<Reservation> findByUserId(UUID userId);
}
```

2. **AMQP Event Publisher**:

```java
@Component
public class AmqpEventPublisher implements DomainEventPublisher {
    private final RabbitTemplate rabbitTemplate;
    
    @Override
    public void publish(DomainEvent event) {
        rabbitTemplate.convertAndSend(
            "book-events", 
            "book.available", 
            event
        );
    }
}
```

## Configuration

Here's how we wire everything together in our Spring Boot application:

```java
/**
 * Application configuration class that handles framework-specific concerns.
 * This is where we:
 * 1. Wire up our domain components
 * 2. Configure transaction management
 * 3. Set up any other infrastructure concerns
 * 
 * By keeping this separate from our domain code, we maintain the ability to:
 * - Switch frameworks if needed
 * - Test domain logic without framework dependencies
 * - Keep domain code focused on business rules
 */
@Configuration
@EnableAspectJAutoProxy
public class ApplicationConfig {
    @Bean
    public BookReservationService bookReservationService(
        BookRepository bookRepository,
        ReservationRepository reservationRepository,
        DomainEventPublisher eventPublisher) {
        return new BookReservationService(
            bookRepository,
            reservationRepository,
            eventPublisher
        );
    }
    
    @Bean
    public TransactionalAspect transactionalAspect(PlatformTransactionManager transactionManager) {
        return new TransactionalAspect(transactionManager);
    }
}

/**
 * Aspect that handles transaction management for our domain services.
 * This approach:
 * 1. Keeps transaction management concerns out of our domain code
 * 2. Makes transaction behavior configurable without touching domain logic
 * 3. Allows us to change transaction management strategy without affecting business rules
 */
@Aspect
@Component
public class TransactionalAspect {
    private final PlatformTransactionManager transactionManager;
    
    public TransactionalAspect(PlatformTransactionManager transactionManager) {
        this.transactionManager = transactionManager;
    }
    
    @Around("@annotation(org.springframework.transaction.annotation.Transactional)")
    public Object manageTransaction(ProceedingJoinPoint pjp) throws Throwable {
        TransactionTemplate transactionTemplate = new TransactionTemplate(transactionManager);
        return transactionTemplate.execute(status -> {
            try {
                return pjp.proceed();
            } catch (Throwable e) {
                throw new RuntimeException(e);
            }
        });
    }
}

@SpringBootApplication
public class LibraryApplication {
    public static void main(String[] args) {
        SpringApplication.run(LibraryApplication.class, args);
    }
}
```

## Benefits of This Architecture

1. **Separation of Concerns**: The core business logic is isolated from infrastructure concerns.
2. **Testability**: We can easily test the domain logic without dealing with HTTP or messaging concerns.
3. **Flexibility**: We can swap out implementations (e.g., change from JPA to MongoDB) without affecting the core logic.
4. **Maintainability**: Changes in one part of the system (e.g., REST API) don't affect other parts.

## Testing Strategy

We can test our system at different levels, taking advantage of our clean architecture:

1. **Domain Logic Tests (Pure Unit Tests)**:
```java
/**
 * Tests for the core domain logic, completely independent of any framework.
 * These tests are fast, reliable, and focus purely on business rules.
 */
public class BookReservationServiceTest {
    private BookReservationService service;
    private BookRepository bookRepository;
    private ReservationRepository reservationRepository;
    private DomainEventPublisher eventPublisher;
    
    @BeforeEach
    void setUp() {
        // Using mocks to isolate the domain logic
        bookRepository = mock(BookRepository.class);
        reservationRepository = mock(ReservationRepository.class);
        eventPublisher = mock(DomainEventPublisher.class);
        
        service = new BookReservationService(
            bookRepository,
            reservationRepository,
            eventPublisher
        );
    }
    
    @Test
    void shouldMakeReservationWhenBookIsAvailable() {
        // Arrange
        UUID bookId = UUID.randomUUID();
        UUID userId = UUID.randomUUID();
        Book availableBook = new Book(bookId, "Clean Code", "123456789");
        availableBook.setStatus(BookStatus.AVAILABLE);
        
        when(bookRepository.findById(bookId))
            .thenReturn(Optional.of(availableBook));
        when(reservationRepository.save(any()))
            .thenAnswer(invocation -> invocation.getArgument(0));
            
        // Act
        Reservation reservation = service.makeReservation(bookId, userId);
        
        // Assert
        assertThat(reservation.getBookId()).isEqualTo(bookId);
        assertThat(reservation.getUserId()).isEqualTo(userId);
        assertThat(availableBook.getStatus()).isEqualTo(BookStatus.RESERVED);
        verify(reservationRepository).save(any(Reservation.class));
    }
    
    @Test
    void shouldThrowWhenBookIsNotAvailable() {
        // Arrange
        UUID bookId = UUID.randomUUID();
        Book reservedBook = new Book(bookId, "Clean Code", "123456789");
        reservedBook.setStatus(BookStatus.RESERVED);
        
        when(bookRepository.findById(bookId))
            .thenReturn(Optional.of(reservedBook));
            
        // Act & Assert
        assertThatThrownBy(() -> service.makeReservation(bookId, UUID.randomUUID()))
            .isInstanceOf(BookNotAvailableException.class);
    }
}

/**
 * Tests for the AMQP adapter implementation.
 * These tests verify that our infrastructure layer correctly handles
 * the translation between AMQP messages and domain events.
 */
public class AmqpEventPublisherTest {
    private AmqpEventPublisher publisher;
    private RabbitTemplate rabbitTemplate;
    
    @BeforeEach
    void setUp() {
        rabbitTemplate = mock(RabbitTemplate.class);
        publisher = new AmqpEventPublisher(rabbitTemplate);
    }
    
    @Test
    void shouldPublishBookAvailableEvent() {
        // Arrange
        UUID bookId = UUID.randomUUID();
        BookAvailableEvent event = new BookAvailableEvent(bookId);
        
        // Act
        publisher.publish(event);
        
        // Assert
        verify(rabbitTemplate).convertAndSend(
            eq("book-events"),
            eq("book.available"),
            argThat(message -> 
                message instanceof BookAvailableMessage &&
                ((BookAvailableMessage) message).getBookId().equals(bookId)
            )
        );
    }
}
```

2. **Integration Tests**:
```java
/**
 * Integration tests that verify the interaction between our adapters
 * and the core domain logic. These tests use Spring's test support
 * but focus on testing our application's behavior rather than
 * framework configuration.
 */
@SpringBootTest
@Transactional
public class ReservationIntegrationTest {
    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private BookRepository bookRepository;
    
    @Autowired
    private ReservationRepository reservationRepository;
    
    /**
     * Note: @SpringBootTest does not provide transactional support by default.
     * We add @Transactional at the class level to ensure test isolation and
     * automatic rollback of database changes after each test method.
     */
    @Test
    void shouldCreateReservationViaRestApi() throws Exception {
        // Arrange
        UUID bookId = UUID.randomUUID();
        UUID userId = UUID.randomUUID();
        
        Book book = new Book(bookId, "Clean Code", "123456789");
        book.setStatus(BookStatus.AVAILABLE);
        bookRepository.save(book);
        
        // Act
        mockMvc.perform(post("/api/reservations")
            .contentType(MediaType.APPLICATION_JSON)
            .content("""
                {
                    "bookId": "%s",
                    "userId": "%s"
                }
                """.formatted(bookId, userId)))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.bookId").value(bookId.toString()))
            .andExpect(jsonPath("$.userId").value(userId.toString()));
            
        // Assert
        List<Reservation> reservations = reservationRepository.findByUserId(userId);
        assertThat(reservations).hasSize(1);
        assertThat(reservations.get(0).getBookId()).isEqualTo(bookId);
        
        Book updatedBook = bookRepository.findById(bookId).orElseThrow();
        assertThat(updatedBook.getStatus()).isEqualTo(BookStatus.RESERVED);
    }
}

@SpringBootTest
@Transactional
public class BookAvailabilityIntegrationTest {
    @Autowired
    private BookRepository bookRepository;
    
    @Autowired
    private RabbitTemplate rabbitTemplate;
    
    @Test
    void shouldProcessBookAvailableMessage() throws Exception {
        // Arrange
        UUID bookId = UUID.randomUUID();
        Book book = new Book(bookId, "Clean Code", "123456789");
        book.setStatus(BookStatus.CHECKED_OUT);
        bookRepository.save(book);
        
        // Act
        rabbitTemplate.convertAndSend(
            "book-availability",
            new BookAvailableMessage(bookId)
        );
        
        // Assert
        await()
            .atMost(5, SECONDS)
            .untilAsserted(() -> {
                Book updatedBook = bookRepository.findById(bookId).orElseThrow();
                assertThat(updatedBook.getStatus()).isEqualTo(BookStatus.AVAILABLE);
            });
            
        // Wait for and verify the actual domain event message
        await()
            .atMost(5, SECONDS)
            .untilAsserted(() -> {
                Message message = rabbitTemplate.receive("book-events", 1000);
                assertThat(message).isNotNull();
                BookAvailableMessage event = (BookAvailableMessage) rabbitTemplate.getMessageConverter()
                    .fromMessage(message);
                assertThat(event.getBookId()).isEqualTo(bookId);
            });
    }
}

/**
 * Test configuration that sets up a test-specific RabbitMQ exchange and queue
 * for capturing domain events.
 */
@Configuration
public class TestRabbitConfig {
    @Bean
    public Queue testBookEventsQueue() {
        return new Queue("book-events", true);
    }
    
    @Bean
    public DirectExchange testBookEventsExchange() {
        return new DirectExchange("book-events");
    }
    
    @Bean
    public Binding testBookEventsBinding(Queue testBookEventsQueue, DirectExchange testBookEventsExchange) {
        return BindingBuilder.bind(testBookEventsQueue)
            .to(testBookEventsExchange)
            .with("book.available");
    }
}