---
layout: post
title: Role Explosion
published: true
categories: 
            - Spring
            - RBAC

---

# Motivation
Sometimes we need to interrogate a lot of roles to ensure that the user has the right permissions to perform an action.

For example in this spring code:

```java
@PostMapping("/users")
@PreAuthorize("hasRole('ADMIN') or hasRole('MANAGER')")
public User doAction(@PathVariable Long id) {
    return myService.doAction(id);
}
```

What if we wanted to add 2 more roles to the `doAction` method?

```java
@PostMapping("/users")
@PreAuthorize("hasRole('ADMIN') or hasRole('MANAGER') or hasRole('SUPERVISOR')")
public User doAction(@PathVariable Long id) {
    return myService.doAction(id);
}
```

You suddenly have 4 roles to interrogate.

## Custom security expression

We can create a custom security expression to handle this.

```java
private boolean hasPrivilege(Authentication authentication, String tagetType, String permission ) {
    return authentication.getAuthorities().stream()
        .anyMatch(grantedAuthority -> grantedAuthority.getAuthority().equals(tagetType + ":" + permission));
}
```

The privelages are stored in the database as a string in the format of `targetType:permission`.


## Database-Driven Privileges

To use custom security expressions with `@PreAuthorize`, we need to create a security expression handler:

```java
@Component
public class CustomSecurityExpressionRoot extends SecurityExpressionRoot implements MethodSecurityExpressionOperations {
    @Autowired
    private PrivilegeRepository privilegeRepository;
    
    private Object filterObject;
    private Object returnObject;
    
    public CustomSecurityExpressionRoot(Authentication authentication) {
        super(authentication);
    }

    public boolean hasPrivilege(String privilege) {
        UserDetails userDetails = (UserDetails) authentication.getPrincipal();
        return privilegeRepository.hasPrivilege(userDetails.getUsername(), privilege);
    }

    @Override
    public void setFilterObject(Object filterObject) { this.filterObject = filterObject; }

    @Override
    public Object getFilterObject() { return filterObject; }

    @Override
    public void setReturnObject(Object returnObject) { this.returnObject = returnObject; }

    @Override
    public Object getReturnObject() { return returnObject; }
}

@Component
public class CustomMethodSecurityExpressionHandler extends DefaultMethodSecurityExpressionHandler {
    @Autowired
    private CustomSecurityExpressionRoot customSecurityExpressionRoot;

    @Override
    protected MethodSecurityExpressionOperations createSecurityExpressionRoot(Authentication authentication, MethodInvocation invocation) {
        CustomSecurityExpressionRoot root = new CustomSecurityExpressionRoot(authentication);
        root.setPermissionEvaluator(getPermissionEvaluator());
        root.setTrustResolver(new AuthenticationTrustResolverImpl());
        return root;
    }
}

@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class MethodSecurityConfig extends GlobalMethodSecurityConfiguration {
    @Autowired
    private CustomMethodSecurityExpressionHandler expressionHandler;

    @Override
    protected MethodSecurityExpressionHandler createExpressionHandler() {
        return expressionHandler;
    }
}
```

Now you can use the custom `hasPrivilege` method in your `@PreAuthorize` annotations:

```java
@PostMapping("/users")
@PreAuthorize("hasPrivilege('DO_ACTION_PRIVILEGE')")
public User doAction(@PathVariable Long id) {
    return myService.doAction(id);
}
```

The `hasPrivilege` method will check the database for the user's privileges, making your permission system dynamic and database-driven.


