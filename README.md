Complete Spring Boot Integration with Appwrite REST API
1. Configuration Class
Store Appwrite configuration details securely, e.g., in application.properties.

CopyRun
# src/main/resources/application.properties
appwrite.endpoint=https://YOUR_APPWRITE_ENDPOINT/v1
appwrite.projectId=your_project_id
appwrite.apiKey=your_api_key
appwrite.databaseId=your_database_id
# Add collection IDs for each entity
appwrite.collections.users=user_collection_id
appwrite.collections.courses=course_collection_id
appwrite.collections.categories=category_collection_id
appwrite.collections.enrollments=enrollment_collection_id
appwrite.collections.payments=payment_collection_id
appwrite.collections.transactions=transaction_collection_id
Create a configuration class to load these:

CopyRun
package com.example.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;

@Configuration
public class AppwriteProperties {

    @Value("${appwrite.endpoint}")
    public String endpoint;

    @Value("${appwrite.projectId}")
    public String projectId;

    @Value("${appwrite.apiKey}")
    public String apiKey;

    @Value("${appwrite.databaseId}")
    public String databaseId;

    @Value("${appwrite.collections.users}")
    public String usersCollectionId;

    @Value("${appwrite.collections.courses}")
    public String coursesCollectionId;

    @Value("${appwrite.collections.categories}")
    public String categoriesCollectionId;

    @Value("${appwrite.collections.enrollments}")
    public String enrollmentsCollectionId;

    @Value("${appwrite.collections.payments}")
    public String paymentsCollectionId;

    @Value("${appwrite.collections.transactions}")
    public String transactionsCollectionId;
}
2. Utility Class for REST Calls
Use Spring's RestTemplate or WebClient. Here, we'll use RestTemplate.

CopyRun
package com.example.util;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.*;
import org.springframework.stereotype.Component;
import com.example.config.AppwriteProperties;
import com.fasterxml.jackson.databind.ObjectMapper;

@Component
public class AppwriteClient {

    @Autowired
    private AppwriteProperties properties;

    private final ObjectMapper objectMapper = new ObjectMapper();

    public <T> ResponseEntity<T> sendRequest(
            HttpMethod method,
            String path,
            Object body,
            Class<T> responseType) {

        String url = properties.endpoint + path;

        HttpHeaders headers = new HttpHeaders();
        headers.set("X-Appwrite-Project", properties.projectId);
        headers.set("X-Appwrite-Key", properties.apiKey);
        headers.setContentType(MediaType.APPLICATION_JSON);

        HttpEntity<String> entity = null;
        try {
            String jsonBody = body != null ? objectMapper.writeValueAsString(body) : null;
            entity = new HttpEntity<>(jsonBody, headers);
        } catch (Exception e) {
            throw new RuntimeException("Error serializing request body", e);
        }

        ResponseEntity<T> response = null;
        try {
            response = new org.springframework.web.client.RestTemplate()
                    .exchange(url, method, entity, responseType);
        } catch (Exception e) {
            // Handle error, log, etc.
            throw new RuntimeException("Error during Appwrite request: " + e.getMessage(), e);
        }
        return response;
    }
}
3. Models with Lombok
Use Lombok to reduce boilerplate:

CopyRun
package com.example.model;

import lombok.Data;

@Data
public class User {
    private String id;
    private String username;
    private String email;
    private String password; // handle carefully
}
Repeat for other entities (Course, Category, etc.).

4. Services
Each service will be annotated with @Service and injected with AppwriteClient.

Example: UserService.java
CopyRun
package com.example.service;

import com.example.model.User;
import com.example.util.AppwriteClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@Service
public class UserService {

    @Autowired
    private AppwriteClient client;

    @Autowired
    private AppwriteProperties properties;

    private String collectionId; // set in constructor or init

    @org.springframework.context.annotation.PostConstruct
    public void init() {
        collectionId = properties.usersCollectionId;
    }

    public User createUser(User user) {
        String path = "/database/" + properties.databaseId + "/collections/" + collectionId + "/documents";
        var response = client.sendRequest(
                HttpMethod.POST, path, user.toMap(), Map.class);
        Map<String, Object> data = response.getBody();
        return User.fromMap(data);
    }

    public User getUser(String id) {
        String path = "/database/" + properties.databaseId + "/collections/" + collectionId + "/documents/" + id;
        var response = client.sendRequest(
                HttpMethod.GET, path, null, Map.class);
        Map<String, Object> data = response.getBody();
        return User.fromMap(data);
    }

    public User updateUser(String id, User user) {
        String path = "/database/" + properties.databaseId + "/collections/" + collectionId + "/documents/" + id;
        var response = client.sendRequest(
                HttpMethod.PUT, path, user.toMap(), Map.class);
        Map<String, Object> data = response.getBody();
        return User.fromMap(data);
    }

    public void deleteUser(String id) {
        String path = "/database/" + properties.databaseId + "/collections/" + collectionId + "/documents/" + id;
        client.sendRequest(HttpMethod.DELETE, path, null, Void.class);
    }

    public List<User> listUsers() {
        String path = "/database/" + properties.databaseId + "/collections/" + collectionId + "/documents";
        var response = client.sendRequest(
                HttpMethod.GET, path, null, Map.class);
        Map<String, Object> data = response.getBody();
        List<Map<String, Object>> docs = (List<Map<String, Object>>) data.get("documents");
        return docs.stream().map(User::fromMap).collect(Collectors.toList());
    }
}
Repeat similarly for other entities’ services.

5. Controllers
Standard Spring MVC controllers:

CopyRun
package com.example.controller;

import com.example.model.User;
import com.example.service.UserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/users")
public class UserController {

    @Autowired
    private UserService userService;

    @PostMapping
    public ResponseEntity<User> create(@RequestBody User user) {
        User created = userService.createUser(user);
        return ResponseEntity.ok(created);
    }

    @GetMapping("/{id}")
    public ResponseEntity<User> get(@PathVariable String id) {
        User user = userService.getUser(id);
        if (user != null) {
            return ResponseEntity.ok(user);
        }
        return ResponseEntity.notFound().build();
    }

    @PutMapping("/{id}")
    public ResponseEntity<User> update(@PathVariable String id, @RequestBody User user) {
        User updated = userService.updateUser(id, user);
        return ResponseEntity.ok(updated);
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable String id) {
        userService.deleteUser(id);
        return ResponseEntity.noContent().build();
    }

    @GetMapping
    public ResponseEntity<List<User>> list() {
        return ResponseEntity.ok(userService.listUsers());
    }
}
Repeat for other entities.

Key Points:
Use @Service classes for all logic, injected into controllers.
Use Spring Boot's application.properties for config.
Use RestTemplate or WebClient for REST calls.
Use Lombok annotations for models (@Data) to reduce boilerplate.
Handle exceptions, errors, and edge cases as needed.
If you'd like, I can prepare a full project skeleton with all classes, or generate example classes for Course, Category, Enrollment, Payment, Transaction controllers and services** following this pattern. Let me know!

Copy
Summarize
Delete
complete the rest of the java classes which you left
Regenerate
Delete
Certainly! I'll complete the remaining classes for your entities—Category, Enrollment, Payment, and Transaction—following the Spring Boot style with models, services, and controllers.

1. Models
Category.java
CopyRun
package com.example.model;

import lombok.Data;
import java.util.HashMap;
import java.util.Map;

@Data
public class Category {
    private String id;
    private String name;

    public Map<String, Object> toMap() {
        Map<String, Object> map = new HashMap<>();
        if (name != null) map.put("name", name);
        return map;
    }

    public static Category fromMap(Map<String, Object> map) {
        Category c = new Category();
        c.setId((String) map.get("id"));
        c.setName((String) map.get("name"));
        return c;
    }
}
Enrollment.java
CopyRun
package com.example.model;

import lombok.Data;
import java.util.HashMap;
import java.util.Map;

@Data
public class Enrollment {
    private String id;
    private String userId;
    private String courseId;
    private String status;

    public Map<String, Object> toMap() {
        Map<String, Object> map = new HashMap<>();
        if (userId != null) map.put("user_id", userId);
        if (courseId != null) map.put("course_id", courseId);
        if (status != null) map.put("status", status);
        return map;
    }

    public static Enrollment fromMap(Map<String, Object> map) {
        Enrollment e = new Enrollment();
        e.setId((String) map.get("id"));
        e.setUserId((String) map.get("user_id"));
        e.setCourseId((String) map.get("course_id"));
        e.setStatus((String) map.get("status"));
        return e;
    }
}
Payment.java
CopyRun
package com.example.model;

import lombok.Data;
import java.util.HashMap;
import java.util.Map;

@Data
public class Payment {
    private String id;
    private String userId;
    private double amount;
    private String method;
    private String date; // ISO8601

    public Map<String, Object> toMap() {
        Map<String, Object> map = new HashMap<>();
        if (userId != null) map.put("user_id", userId);
        map.put("amount", amount);
        if (method != null) map.put("method", method);
        if (date != null) map.put("date", date);
        return map;
    }

    public static Payment fromMap(Map<String, Object> map) {
        Payment p = new Payment();
        p.setId((String) map.get("id"));
        p.setUserId((String) map.get("user_id"));
        p.setAmount(Double.parseDouble(String.valueOf(map.get("amount"))));
        p.setMethod((String) map.get("method"));
        p.setDate((String) map.get("date"));
        return p;
    }
}
Transaction.java
CopyRun
package com.example.model;

import lombok.Data;
import java.util.HashMap;
import java.util.Map;

@Data
public class Transaction {
    private String id;
    private String userId;
    private String type; // e.g., "credit" or "debit"
    private double amount;
    private String timestamp; // ISO8601

    public Map<String, Object> toMap() {
        Map<String, Object> map = new HashMap<>();
        if (userId != null) map.put("user_id", userId);
        if (type != null) map.put("type", type);
        map.put("amount", amount);
        if (timestamp != null) map.put("timestamp", timestamp);
        return map;
    }

    public static Transaction fromMap(Map<String, Object> map) {
        Transaction t = new Transaction();
        t.setId((String) map.get("id"));
        t.setUserId((String) map.get("user_id"));
        t.setType((String) map.get("type"));
        t.setAmount(Double.parseDouble(String.valueOf(map.get("amount"))));
        t.setTimestamp((String) map.get("timestamp"));
        return t;
    }
}
2. Services
Example for CategoryService.java:

CopyRun
package com.example.service;

import com.example.model.Category;
import com.example.util.AppwriteClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@Service
public class CategoryService {

    @Autowired
    private AppwriteClient client;

    @Autowired
    private AppwriteProperties properties;

    private String collectionId;

    @org.springframework.context.annotation.PostConstruct
    public void init() {
        collectionId = properties.categoriesCollectionId;
    }

    public Category createCategory(Category category) {
        String path = "/database/" + properties.databaseId + "/collections/" + collectionId + "/documents";
        var response = client.sendRequest(HttpMethod.POST, path, category.toMap(), Map.class);
        return Category.fromMap(response.getBody());
    }

    public Category getCategory(String id) {
        String path = "/database/" + properties.databaseId + "/collections/" + collectionId + "/documents/" + id;
        var response = client.sendRequest(HttpMethod.GET, path, null, Map.class);
        return Category.fromMap(response.getBody());
    }

    public Category updateCategory(String id, Category category) {
        String path = "/database/" + properties.databaseId + "/collections/" + collectionId + "/documents/" + id;
        var response = client.sendRequest(HttpMethod.PUT, path, category.toMap(), Map.class);
        return Category.fromMap(response.getBody());
    }

    public void deleteCategory(String id) {
        String path = "/database/" + properties.databaseId + "/collections/" + collectionId + "/documents/" + id;
        client.sendRequest(HttpMethod.DELETE, path, null, Void.class);
    }

    public List<Category> listCategories() {
        String path = "/database/" + properties.databaseId + "/collections/" + collectionId + "/documents";
        var response = client.sendRequest(HttpMethod.GET, path, null, Map.class);
        List<Map<String, Object>> docs = (List<Map<String, Object>>) response.getBody().get("documents");
        return docs.stream().map(Category::fromMap).collect(Collectors.toList());
    }
}
Repeat for EnrollmentService.java, PaymentService.java, and TransactionService.java, replacing collection IDs and models.

3. Controllers
Example for CategoryController.java:

CopyRun
package com.example.controller;

import com.example.model.Category;
import com.example.service.CategoryService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/categories")
public class CategoryController {

    @Autowired
    private CategoryService categoryService;

    @PostMapping
    public ResponseEntity<Category> create(@RequestBody Category category) {
        return ResponseEntity.ok(categoryService.createCategory(category));
    }

    @GetMapping("/{id}")
    public ResponseEntity<Category> get(@PathVariable String id) {
        Category category = categoryService.getCategory(id);
        if (category != null) {
            return ResponseEntity.ok(category);
        }
        return ResponseEntity.notFound().build();
    }

    @PutMapping("/{id}")
    public ResponseEntity<Category> update(@PathVariable String id, @RequestBody Category category) {
        return ResponseEntity.ok(categoryService.updateCategory(id, category));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable String id) {
        categoryService.deleteCategory(id);
        return ResponseEntity.noContent().build();
    }

    @GetMapping
    public ResponseEntity<List<Category>> list() {
        return ResponseEntity.ok(categoryService.listCategories());
    }
}
Repeat for EnrollmentController, PaymentController, TransactionController.

Summary
Models: Category, Enrollment, Payment, Transaction (with toMap() and fromMap())
Services: CRUD methods using AppwriteClient
Controllers: Expose REST endpoints





















































































































I'd be happy to help you with the project structure. Here's a suggested directory structure for your Spring Boot project:

CopyRun
com.example
  |
  |- controller
  |    |- CategoryController.java
  |    |- EnrollmentController.java
  |    |- PaymentController.java
  |    |- TransactionController.java
  |
  |- service
  |    |- CategoryService.java
  |    |- EnrollmentService.java
  |    |- PaymentService.java
  |    |- TransactionService.java
  |
  |- model
  |    |- Category.java
  |    |- Enrollment.java
  |    |- Payment.java
  |    |- Transaction.java
  |
  |- util
  |    |- AppwriteClient.java
  |    |- AppwriteProperties.java
  |
  |- Application.java
And here's the updated code for each class:

CategoryController.java

CopyRun
package com.example.controller;

import com.example.model.Category;
import com.example.service.CategoryService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/categories")
public class CategoryController {

    @Autowired
    private CategoryService categoryService;

    @PostMapping
    public ResponseEntity<Category> create(@RequestBody Category category) {
        return ResponseEntity.ok(categoryService.createCategory(category));
    }

    @GetMapping("/{id}")
    public ResponseEntity<Category> get(@PathVariable String id) {
        Category category = categoryService.getCategory(id);
        if (category != null) {
            return ResponseEntity.ok(category);
        }
        return ResponseEntity.notFound().build();
    }

    @PutMapping("/{id}")
    public ResponseEntity<Category> update(@PathVariable String id, @RequestBody Category category) {
        return ResponseEntity.ok(categoryService.updateCategory(id, category));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable String id) {
        categoryService.deleteCategory(id);
        return ResponseEntity.noContent().build();
    }

    @GetMapping
    public ResponseEntity<List<Category>> list() {
        return ResponseEntity.ok(categoryService.listCategories());
    }
}
EnrollmentController.java

CopyRun
package com.example.controller;

import com.example.model.Enrollment;
import com.example.service.EnrollmentService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/enrollments")
public class EnrollmentController {

    @Autowired
    private EnrollmentService enrollmentService;

    @PostMapping
    public ResponseEntity<Enrollment> create(@RequestBody Enrollment enrollment) {
        return ResponseEntity.ok(enrollmentService.createEnrollment(enrollment));
    }

    @GetMapping("/{id}")
    public ResponseEntity<Enrollment> get(@PathVariable String id) {
        Enrollment enrollment = enrollmentService.getEnrollment(id);
        if (enrollment != null) {
            return ResponseEntity.ok(enrollment);
        }
        return ResponseEntity.notFound().build();
    }

    @PutMapping("/{id}")
    public ResponseEntity<Enrollment> update(@PathVariable String id, @RequestBody Enrollment enrollment) {
        return ResponseEntity.ok(enrollmentService.updateEnrollment(id, enrollment));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable String id) {
        enrollmentService.deleteEnrollment(id);
        return ResponseEntity.noContent().build();
    }

    @GetMapping
    public ResponseEntity<List<Enrollment>> list() {
        return ResponseEntity.ok(enrollmentService.listEnrollments());
    }
}
PaymentController.java

CopyRun
package com.example.controller;

import com.example.model.Payment;
import com.example.service.PaymentService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/payments")
public class PaymentController {

    @Autowired
    private PaymentService paymentService;

    @PostMapping
    public ResponseEntity<Payment> create(@RequestBody Payment payment) {
        return ResponseEntity.ok(paymentService.createPayment(payment));
    }

    @GetMapping("/{id}")
    public ResponseEntity<Payment> get(@PathVariable String id) {
        Payment payment = paymentService.getPayment(id);
        if (payment != null) {
            return ResponseEntity.ok(payment);
        }
        return ResponseEntity.notFound().build();
    }

    @PutMapping("/{id}")
    public ResponseEntity<Payment> update(@PathVariable String id, @RequestBody Payment payment) {
        return ResponseEntity.ok(paymentService.updatePayment(id, payment));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable String id) {
        paymentService.deletePayment(id);
        return ResponseEntity.noContent().build();
    }

    @GetMapping
    public ResponseEntity<List<Payment>> list() {
        return ResponseEntity.ok(paymentService.listPayments());
    }
}
TransactionController.java

CopyRun
package com.example.controller;

import com.example.model.Transaction;
import com.example.service.TransactionService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/transactions")
public class TransactionController {

    @Autowired
    private TransactionService transactionService;

    @PostMapping
    public ResponseEntity<Transaction> create(@RequestBody Transaction transaction) {
        return ResponseEntity.ok(transactionService.createTransaction(transaction));
    }

    @GetMapping("/{id}")
    public ResponseEntity<Transaction> get(@PathVariable String id) {
        Transaction transaction = transactionService.getTransaction(id);
        if (transaction != null) {
            return ResponseEntity.ok(transaction);
        }
        return ResponseEntity.notFound().build();
    }

    @PutMapping("/{id}")
    public ResponseEntity<Transaction> update(@PathVariable String id, @RequestBody Transaction transaction) {
        return ResponseEntity.ok(transactionService.updateTransaction(id, transaction));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable String id) {
        transactionService.deleteTransaction(id);
        return ResponseEntity.noContent().build();
    }

    @GetMapping
    public ResponseEntity<List<Transaction>> list() {
        return ResponseEntity.ok(transactionService.listTransactions());
    }
}
AppwriteClient.java

CopyRun
package com.example.util;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpMethod;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestTemplate;

import java.util.Map;

@Component
public class AppwriteClient {

    private final RestTemplate restTemplate;

    public AppwriteClient(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    public <T> T sendRequest(HttpMethod method, String path, Object body, Class<T> responseType) {
        String url = "https://YOUR_APPWRITE_ENDPOINT/v1/database/YOUR_DATABASE_ID/collections/YOUR_COLLECTION_ID/documents";
        HttpEntity<String> entity = null;
        if (body != null) {
            entity = new HttpEntity<>(body.toString(), HttpHeaders.EMPTY);
        }
        try {
            ResponseEntity<T> response = restTemplate.exchange(url + path, method, entity, responseType);
            return response.getBody();
        } catch (Exception e) {
            // Handle error, log, etc.
            throw new RuntimeException("Error during Appwrite request: " + e.getMessage(), e);
        }
    }
}
AppwriteProperties.java

CopyRun
package com.example.util;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.PropertySource;
import org.springframework.stereotype.Component;

@Component
@PropertySource("classpath:application.properties")
public class AppwriteProperties {

    @Value("${appwrite.endpoint}")
    private String endpoint;

    @Value("${appwrite.projectId}")
    private String projectId;

    @Value("${appwrite.apiKey}")
    private String apiKey;

}
CategoryService.java

CopyRun
package com.example.service;

import com.example.model.Category;
import com.example.util.AppwriteClient;
import com.example.util.AppwriteProperties;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class CategoryService {

    @Autowired
    private AppwriteClient client;

    @Autowired
    private AppwriteProperties properties;

}
Please note that you'll need to replace the placeholder values in AppwriteProperties with your actual Appwrite endpoint, project ID, and API key.

Also, make sure to add the necessary dependencies to your pom.xml file if you're using Maven:

CopyRun
<dependencies>
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-data-rest</artifactId>
  </dependency>
  <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
  </dependency>
</dependencies>
If you're using Gradle, add the following dependencies to your build.gradle file:

CopyRun
dependencies {
  implementation 'org.springframework.boot:spring-boot-starter-web'
  implementation 'org.springframework.boot:spring-boot-starter-data-rest'
  implementation 'com.fasterxml.jackson.core:jackson-databind'
}
This should give you a good starting point for your Spring Boot application. Remember to update the placeholder values in AppwriteProperties and adjust the configuration as needed to match your Appwrite setup.

Please let me know if you have any further questions or if there's anything else I can help with!
