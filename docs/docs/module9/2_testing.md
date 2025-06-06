---
sidebar_position: 1
---

# Rust's Testing Philosophy

Rust has a built-in testing framework that's simple yet powerful. The testing philosophy is:

1. **Tests live close to the code they test**: Unit tests often live in the same file as the implementation
2. **No separate test framework needed**: Tests run with the `cargo test` command
3. **First-class support in the language**: Testing is a core feature of Rust, not an afterthought
4. **Compile-time guarantees reduce need for some tests**: Many bugs caught by tests in other languages are caught by the compiler in Rust

## Unit Testing in Rust

Unit tests in Rust typically live in the same file as the code they test, in a special module annotated with `#[cfg(test)]`. This attribute ensures the test code is only compiled when running tests.

Here's an example:

```rust showLineNumbers
// Implementation code
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

// Test module
#[cfg(test)]
mod tests {
    // Import parent scope
    use super::*;
    
    #[test]
    fn test_add() {
        assert_eq!(add(2, 2), 4);
    }
    
    #[test]
    fn test_add_negative() {
        assert_eq!(add(-1, -2), -3);
    }
}
```

Key points:
- `#[cfg(test)]` ensures test code is only included when testing
- `use super::*` imports all items from the parent module
- `#[test]` marks functions as test cases
- `assert_eq!`, `assert!`, and `assert_ne!` macros help with assertions

## Integration Testing

Integration tests live in a separate directory called `tests` at the root of your project. Each file in this directory is compiled as a separate crate.

```
my_project/
├── src/
│   └── lib.rs
└── tests/
    ├── api_tests.rs
    └── data_tests.rs
```

An integration test might look like:

```rust showLineNumbers
use reqwest::redirect::Policy;
use reqwest::Client;
use uuid::Uuid;

#[tokio::test]
async fn when_a_user_registers_they_should_then_be_able_to_login() {
    let id = Uuid::new_v4();
    let email_under_test = format!("{}@test.com", id);

    let api_endpoint = retrieve_api_endpoint().await;

    let http_client = Client::builder()
        .timeout(std::time::Duration::from_secs(2))
        .redirect(Policy::none())
        .build()
        .unwrap();
    

    let result = http_client
        .post(format!("{}users", api_endpoint))
        .header("Content-Type", "application/json")
        .body(serde_json::json!({"emailAddress": email_under_test, "password": "Testing!23", "name": "James"}).to_string())
        .send()
        .await;

    assert!(result.is_ok());

    let response = result.unwrap();

    assert_eq!(response.status(), 201);

    let login_response = http_client
        .post(format!("{}login", api_endpoint))
        .header("Content-Type", "application/json")
        .body(serde_json::json!({"emailAddress": email_under_test, "password": "Testing!23"}).to_string())
        .send()
        .await;

    assert_eq!(login_response.unwrap().status(), 200);
}

async fn retrieve_api_endpoint() -> String {
    // You could write code here to dynamically retrieve the API endpoint from your environment or configuration.

    "http://localhost:3000/".to_string()
}
```

Here you're using the `reqwest` crate, which gives you a simple HTTP client, to actually make requests to a running instance of your API.

Integration tests:
- Test your code's public API
- Verify that components work together correctly
- Run with the same `cargo test` command

## Testing Async Code

To test async functions, you need to use a runtime like tokio:

```rust showLineNumbers
#[cfg(test)]
mod tests {
    use super::*;
    
    #[tokio::test]
    async fn test_async_function() {
        let result = fetch_data().await;
        assert!(result.is_ok());
    }
}
```

The `#[tokio::test]` attribute sets up the tokio runtime for your async test.

## Test Fixtures

For tests that need similar setup and teardown, you can use Rust's Drop trait:

```rust showLineNumbers
struct TestFixture {
    // Test data and state.
    db_connection: PgPool,
}

impl TestFixture {
    async fn new() -> Self {
        // Setup code.
        let db_connection = PgPool::connect("test_db_url").await.unwrap();
        // Run migrations, seed data.
        
        Self { db_connection }
    }
}

impl Drop for TestFixture {
    fn drop(&mut self) {
        // Cleanup code.
        // This runs when the fixture goes out of scope
    }
}

#[tokio::test]
async fn test_with_fixture() {
    let fixture = TestFixture::new().await;
    
    // Test using fixture.
    let result = get_user(&fixture.db_connection, "test@example.com").await;
    assert!(result.is_ok());
    
    // Fixture is automatically cleaned up when it goes out of scope
}
```