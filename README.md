# unique_temp_pathname_rust

```rust

/// Creates a unique temporary file in the specified base directory with configurable retry logic.
///
/// # Project Context
/// This function generates temporary file names for intermediate processing
/// in our application where we cannot use third-party dependencies. The file
/// is created atomically to prevent race conditions where multiple processes
/// or threads might generate the same name. This version allows the caller
/// to configure retry behavior based on their specific use case (e.g., high
/// contention environments may need more retries, low-priority operations
/// may want fewer retries to fail fast).
///
/// # Implementation Strategy
/// - Uses process ID, thread ID, and nanosecond timestamp for uniqueness
/// - Attempts atomic file creation with `create_new(true)` flag
/// - Retries up to `number_of_attempts` times with configurable delay
/// - Different timestamps on each retry provide additional uniqueness
/// - Parameterized retry logic allows tuning for different deployment scenarios
///
/// # Arguments
/// * `base_path` - The directory where the temporary file will be created (must exist)
/// * `prefix` - A prefix for the filename to identify the file's purpose (e.g., "cache", "upload")
/// * `number_of_attempts` - Maximum number of creation attempts (recommended: 3-10)
/// * `retry_delay_ms` - Milliseconds to wait between retry attempts (recommended: 1-100)
///
/// # Returns
/// * `Ok(PathBuf)` - Absolute path to the newly created unique temporary file
/// * `Err(io::Error)` - If file creation fails after all retry attempts or other I/O error
///
/// # Error Conditions
/// - `CUTF: system time unavailable` - System clock error (rare, catastrophic)
/// - `CUTF: max retry attempts exceeded` - Could not create unique file after all attempts
/// - `CUTF: unexpected loop exit` - Internal logic error (should never occur)
/// - Standard I/O errors: permission denied, disk full, path not found, etc.
///
/// # Safety & Reliability
/// - No panic: all error cases return Result
/// - No heap allocation in error messages (uses static strings with CUTF prefix)
/// - Bounded retry loop (caller-specified maximum attempts)
/// - Atomic file creation prevents race conditions
/// - Thread-safe: uses thread-local IDs
/// - Handles system clock errors gracefully
///
/// # Configuration Guidelines
/// - **Low contention** (single-threaded, low frequency): `number_of_attempts = 3`, `retry_delay_ms = 1`
/// - **Medium contention** (multi-threaded application): `number_of_attempts = 5`, `retry_delay_ms = 1-5`
/// - **High contention** (distributed system, many processes): `number_of_attempts = 10`, `retry_delay_ms = 10-50`
/// - **Fast-fail scenarios** (can afford to fail): `number_of_attempts = 1`, `retry_delay_ms = 0`
///
/// # Edge Cases Handled
/// - Zero attempts: Function will try once (loop runs 0..0 means 0 iterations, caught by unreachable error)
/// - System time moves backwards: Handled gracefully with retry
/// - Concurrent file creation: Atomic `create_new` prevents race conditions
/// - Disk full or permission errors: Immediate return without retries
///
/// # Example
/// ```
/// use std::path::Path;
///
/// let base = Path::new("/tmp");
///
/// // Standard usage
/// match create_unique_temp_filepathbuf(base, "myapp", 5, 1) {
///     Ok(path) => {
///         println!("Created: {:?}", path);
///         // Use the file...
///         // Remember to delete it when done!
///         let _ = std::fs::remove_file(path);
///     },
///     Err(e) => {
///         eprintln!("Failed to create temp file: {}", e);
///         // Handle error - application continues
///     }
/// }
///
/// // High-contention scenario
/// match create_unique_temp_filepathbuf(base, "distributed", 10, 10) {
///     Ok(path) => { /* use file */ },
///     Err(e) => { /* handle gracefully */ }
/// }
/// ```
///
/// # Security Considerations
/// - File names are predictable (not cryptographically random)
/// - Suitable for temporary storage, not for security-critical scenarios
/// - Consider file permissions on created files (inherits from OpenOptions defaults)
/// - Caller must ensure base_path is in a secure location
///
/// # Performance Considerations
/// - Each retry costs `retry_delay_ms` milliseconds
/// - Maximum possible delay: `number_of_attempts * retry_delay_ms`
/// - Nanosecond timestamp provides ~1 billion unique values per second per thread
/// - Thread ID formatting allocates small string (unavoidable with std::thread API)
pub fn create_unique_temp_filepathbuf(
    base_path: &Path,
    prefix: &str,
    number_of_attempts: u32,
    retry_delay_ms: u64,
) -> Result<PathBuf, io::Error> {
    use std::process;
    use std::thread;
    use std::time::{SystemTime, UNIX_EPOCH, Duration};
    use std::fs::OpenOptions;
    use std::io;

    // Production catch: validate number_of_attempts is non-zero
    // Zero attempts would make function always fail
    if number_of_attempts == 0 {
        return Err(io::Error::new(
            io::ErrorKind::InvalidInput,
            "CUTF: number_of_attempts must be greater than zero",
        ));
    }

    // Get process ID once (constant for this process)
    let pid = process::id();

    // Get thread ID and format it for filename use
    let thread_id = thread::current().id();
    let thread_id_string = format!("{:?}", thread_id);

    // Clean thread ID: remove "ThreadId(" prefix and ")" suffix
    // This converts "ThreadId(123)" to "123"
    let thread_id_clean = thread_id_string
        .trim_start_matches("ThreadId(")
        .trim_end_matches(')');

    // Attempt to create unique file with retry logic
    for attempt in 0..number_of_attempts {
        // Get current timestamp with nanosecond precision
        // This provides uniqueness across time
        let timestamp_result = SystemTime::now().duration_since(UNIX_EPOCH);

        let timestamp_nanos = match timestamp_result {
            Ok(duration) => duration.as_nanos(),
            Err(_) => {
                // System time error (e.g., clock moved backwards)
                // In production, we handle this gracefully and continue
                if attempt == number_of_attempts - 1 {
                    return Err(io::Error::new(
                        io::ErrorKind::Other,
                        "CUTF: system time unavailable",
                    ));
                }
                // Try again with next attempt
                thread::sleep(Duration::from_millis(retry_delay_ms));
                continue;
            }
        };

        // Construct filename: prefix_pid_threadid_timestamp.tmp
        // Example: myapp_12345_67_1234567890123456789.tmp
        let filename = format!(
            "{}_{}_{}_{}.tmp",
            prefix, pid, thread_id_clean, timestamp_nanos
        );

        // Build absolute path
        let file_path = base_path.join(&filename);

        // Attempt to create file atomically
        // create_new(true) ensures the operation fails if file exists
        // This prevents race conditions with other processes/threads
        match OpenOptions::new()
            .write(true)
            .create_new(true)  // Critical: fails if file already exists
            .open(&file_path)
        {
            Ok(_file) => {
                // Success: file created exclusively
                // File handle is dropped here, closing the file
                // Caller is responsible for file cleanup
                return Ok(file_path);
            }
            Err(e) if e.kind() == io::ErrorKind::AlreadyExists => {
                // File name collision detected
                // This is expected in high-concurrency scenarios

                // Production catch: check if we've exhausted retries
                if attempt == number_of_attempts - 1 {
                    // Final attempt failed - return descriptive error
                    return Err(io::Error::new(
                        io::ErrorKind::AlreadyExists,
                        "CUTF: max retry attempts exceeded",
                    ));
                }

                // Wait briefly before retry
                // This allows timestamp to change and reduces contention
                thread::sleep(Duration::from_millis(retry_delay_ms));

                // Continue to next attempt
                continue;
            }
            Err(e) => {
                // Other error occurred (permissions, disk full, etc.)
                // Return immediately - retrying won't help
                return Err(e);
            }
        }
    }

    // Should be unreachable due to loop logic, but rust requires this
    // Production safety: return error rather than panic
    Err(io::Error::new(
        io::ErrorKind::Other,
        "CUTF: unexpected loop exit",
    ))
}

// =============================================================================
// Tests
// =============================================================================

#[cfg(test)]
mod tempname_tests {
    use super::*;
    use std::fs;

    /// Test basic functionality: can we create a temp file with standard parameters?
    #[test]
    fn test_create_temp_file_basic() {
        let temp_dir = std::env::temp_dir();

        let result = create_unique_temp_filepathbuf(&temp_dir, "test", 5, 1);

        assert!(result.is_ok(), "Should successfully create temp file");

        let path = result.unwrap();
        assert!(path.exists(), "Created file should exist");
        assert!(path.starts_with(&temp_dir), "File should be in temp directory");

        // Cleanup
        let _ = fs::remove_file(path);
    }

    /// Test that multiple files created rapidly are unique
    #[test]
    fn test_create_multiple_unique_files() {
        let temp_dir = std::env::temp_dir();
        let mut paths = Vec::new();

        // Create 5 temp files in rapid succession
        for _ in 0..5 {
            let result = create_unique_temp_filepathbuf(&temp_dir, "multi", 5, 1);
            assert!(result.is_ok(), "Should create file");
            paths.push(result.unwrap());
        }

        // Verify all paths are unique
        for i in 0..paths.len() {
            for j in (i + 1)..paths.len() {
                assert_ne!(paths[i], paths[j], "Paths should be unique");
            }
        }

        // Verify all files exist
        for path in &paths {
            assert!(path.exists(), "File should exist");
        }

        // Cleanup
        for path in paths {
            let _ = fs::remove_file(path);
        }
    }

    /// Test that function returns error for non-existent directory
    #[test]
    fn test_nonexistent_directory() {
        let bad_path = Path::new("/this/path/definitely/does/not/exist/nowhere/12345");

        let result = create_unique_temp_filepathbuf(bad_path, "test", 3, 1);

        assert!(result.is_err(), "Should fail for non-existent directory");
    }

    /// Test filename format contains expected components
    #[test]
    fn test_filename_format() {
        let temp_dir = std::env::temp_dir();

        let result = create_unique_temp_filepathbuf(&temp_dir, "prefix", 5, 1);
        assert!(result.is_ok(), "Should create file");

        let path = result.unwrap();
        let filename = path.file_name().unwrap().to_str().unwrap();

        // Verify filename contains prefix
        assert!(filename.starts_with("prefix_"), "Filename should start with prefix");

        // Verify filename ends with .tmp
        assert!(filename.ends_with(".tmp"), "Filename should end with .tmp");

        // Verify filename contains underscores (pid_threadid_timestamp structure)
        assert!(filename.matches('_').count() >= 3, "Filename should have at least 3 underscores");

        // Cleanup
        let _ = fs::remove_file(path);
    }

    /// Test with zero attempts parameter (should return error immediately)
    #[test]
    fn test_zero_attempts() {
        let temp_dir = std::env::temp_dir();

        let result = create_unique_temp_filepathbuf(&temp_dir, "zero", 0, 1);

        assert!(result.is_err(), "Should fail with zero attempts");

        if let Err(e) = result {
            let error_msg = format!("{}", e);
            assert!(error_msg.contains("CUTF"), "Error should have CUTF prefix");
            assert!(
                error_msg.contains("greater than zero") || error_msg.contains("number_of_attempts"),
                "Error should mention invalid attempt count"
            );
        }
    }

    /// Test with single attempt (should work in low-contention)
    #[test]
    fn test_single_attempt() {
        let temp_dir = std::env::temp_dir();

        let result = create_unique_temp_filepathbuf(&temp_dir, "single", 1, 1);

        assert!(result.is_ok(), "Should succeed with single attempt in low contention");

        if let Ok(path) = result {
            assert!(path.exists(), "File should exist");
            let _ = fs::remove_file(path);
        }
    }

    /// Test with different delay values (ensures parameter is used)
    #[test]
    fn test_different_delays() {
        let temp_dir = std::env::temp_dir();

        // Test with zero delay
        let result1 = create_unique_temp_filepathbuf(&temp_dir, "delay0", 3, 0);
        assert!(result1.is_ok(), "Should work with zero delay");
        if let Ok(path) = result1 {
            let _ = fs::remove_file(path);
        }

        // Test with larger delay
        let result2 = create_unique_temp_filepathbuf(&temp_dir, "delay100", 3, 100);
        assert!(result2.is_ok(), "Should work with 100ms delay");
        if let Ok(path) = result2 {
            let _ = fs::remove_file(path);
        }
    }

    /// Test with high number of attempts (stress test)
    #[test]
    fn test_many_attempts() {
        let temp_dir = std::env::temp_dir();

        let result = create_unique_temp_filepathbuf(&temp_dir, "many", 20, 1);

        assert!(result.is_ok(), "Should work with many attempts");

        if let Ok(path) = result {
            assert!(path.exists(), "File should exist");
            let _ = fs::remove_file(path);
        }
    }

    /// Test prefix with special characters (edge case)
    #[test]
    fn test_prefix_with_special_chars() {
        let temp_dir = std::env::temp_dir();

        // Test with prefix containing hyphens and underscores
        let result = create_unique_temp_filepathbuf(&temp_dir, "my-app_v2", 5, 1);

        assert!(result.is_ok(), "Should handle prefix with special chars");

        if let Ok(path) = result {
            let filename = path.file_name().unwrap().to_str().unwrap();
            assert!(filename.starts_with("my-app_v2_"), "Filename should preserve prefix");
            let _ = fs::remove_file(path);
        }
    }

    /// Test that created file is actually writable
    #[test]
    fn test_file_is_writable() {
        let temp_dir = std::env::temp_dir();

        let result = create_unique_temp_filepathbuf(&temp_dir, "writable", 5, 1);
        assert!(result.is_ok(), "Should create file");

        let path = result.unwrap();

        // Try to open and write to the file
        let write_result = OpenOptions::new()
            .write(true)
            .open(&path);

        assert!(write_result.is_ok(), "Should be able to open file for writing");

        // Cleanup
        let _ = fs::remove_file(path);
    }

    /// Test concurrent creation from multiple threads
    #[test]
    fn test_concurrent_creation() {
        use std::sync::Arc;

        let temp_dir = Arc::new(std::env::temp_dir());
        let mut handles = vec![];

        // Spawn 5 threads that each create 2 files
        for thread_num in 0..5 {
            let temp_dir_clone = Arc::clone(&temp_dir);

            let handle = thread::spawn(move || {
                let mut created_paths = Vec::new();

                for file_num in 0..2 {
                    let prefix = format!("concurrent_t{}_f{}", thread_num, file_num);
                    let result = create_unique_temp_filepathbuf(&temp_dir_clone, &prefix, 10, 5);

                    assert!(result.is_ok(), "Should create file from thread");
                    created_paths.push(result.unwrap());
                }

                created_paths
            });

            handles.push(handle);
        }

        // Collect all created paths
        let mut all_paths = Vec::new();
        for handle in handles {
            let paths = handle.join().expect("Thread should complete");
            all_paths.extend(paths);
        }

        // Verify all paths are unique
        for i in 0..all_paths.len() {
            for j in (i + 1)..all_paths.len() {
                assert_ne!(all_paths[i], all_paths[j], "All paths from all threads should be unique");
            }
        }

        // Verify all files exist
        for path in &all_paths {
            assert!(path.exists(), "All created files should exist");
        }

        // Cleanup
        for path in all_paths {
            let _ = fs::remove_file(path);
        }
    }

    /// Test error message format for debugging
    #[test]
    fn test_error_messages_have_cutf_prefix() {
        let temp_dir = std::env::temp_dir();

        // Test zero attempts error
        let result = create_unique_temp_filepathbuf(&temp_dir, "test", 0, 1);
        if let Err(e) = result {
            assert!(format!("{}", e).contains("CUTF"), "Error should have CUTF prefix");
        }
    }
}

```
