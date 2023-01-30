## Code coverage
Code coverage is not a silver bullet for smart contract testing, because code in smart contracts usually exhibit extremely context-dependent behavior and thus obtaining high coverage does not guarantee that all behaviors have been explored. 

However, measuring code coverage can be one of means to evaluate the baseline quality of testcases. So, we provide this tutorial, which explains how to measure of code coverage and how to obtain a coverage report of cosmwasm smart contracts.

## Usage
We use [grcov](https://github.com/mozilla/grcov), which can generate source-based coverage for a Rust project. To obtain a coverage report of cosmwasm smart contracts using `grcov`, we need to extend some code in contracts. 

1. Additional codes, which mean code coverage information are needed in contract's code - `contract.rs`.

   ```
   mod memory {
    use std::convert::TryFrom;
    use std::mem;
    use std::vec::Vec;

    #[repr(C)]
    pub struct Region {
        /// The beginning of the region expressed as bytes from the beginning of the linear memory
        pub offset: u32,
        /// The number of bytes available in this region
        pub capacity: u32,
        /// The number of bytes used in this region
        pub length: u32,
    }

    pub fn release_buffer(buffer: Vec<u8>) -> *mut Region {
        let region = build_region(&buffer);
        mem::forget(buffer);
        Box::into_raw(region)
    }

    pub fn build_region(data: &[u8]) -> Box<Region> {
        let data_ptr = data.as_ptr() as usize;
        build_region_from_components(
            u32::try_from(data_ptr).expect("pointer doesn't fit in u32"),
            u32::try_from(data.len()).expect("length doesn't fit in u32"),
            u32::try_from(data.len()).expect("length doesn't fit in u32"),
        )
    }

    fn build_region_from_components(offset: u32, capacity: u32, length: u32) -> Box<Region> {
        Box::new(Region {
            offset,
            capacity,
            length,
            })
        }
    }

   mod coverage {
    use super::memory::release_buffer;
    use minicov::capture_coverage;
    #[no_mangle]
    extern "C" fn dump_coverage() -> u32 {
        let coverage = capture_coverage();
        release_buffer(coverage) as u32
        }
    }

2. Compile with flags, which enabled LLVM's source-based code coverage instrumentation using `cargo`. Then, artifacts with cov will be made. 
   - `-C instrument-coverage`
3. Use these artifacts in spaceship's python codes.
4. Generate some python codes like below.

    ```
    test-<component_name>
    |_____test.py
    |_____test-with-cov.py
    |_____visualize-cov.py
    |_____artifacts
    |_____artifacts-cov
    ```
   - `test.py` should contain all test case logic. This code should be written using spaceship's Python binding.
   - `test-with-cov.py` is identical to `test.py` except that it contains code coverage information.
   - `visualize-cov.py` is a script used to emit a HTML report from the coverage emitted via `test-with-cov.py`.
   - `artifacts` and `artifacts-cov` contains WASM binaries and native binaries that assist the execution of contracts in `test.py` and `test-with-cov.py`.
   - [example](https://github.com/dream-academy/terraswap-tc)
5. Run `test-with-cov.py` script to generate `cov` directory.

    ```
    python3 test-with-cov.py
    ```
   - `test.py` and `test-with-cov.py` depends on spaceship. We recommend to run in virtualenv.
   - [How to run](https://github.com/pr0cf5/spaceship/tree/main/tutorials/building)
6. Run `visualize-cov.py` script to generate a `grcov` style coverage report.

    ```
    python3 visualize-cov.py <cov_dir> <src_dir>
    ```
    - `visualize-cov.py` makes `cov_report.zip` in <src_dir>.
    - `grcov` provides [various types](https://github.com/mozilla/grcov#alternative-reports) of outputs. Users can set one of the output types provided by `grcov` in `visualize-cov.py`.
    - If user choose `html`, the `index.html` will generate in `cov_report.zip`. Then, user can see that reports in local.
## Example of code coverage report
Belows are example of code coverage reports for terraswap's components.

- [factory](https://procfs-web3.github.io/terraswap-tc-coverage/factory/)
- [pair](https://procfs-web3.github.io/terraswap-tc-coverage/pair/)
- [router](https://procfs-web3.github.io/terraswap-tc-coverage/router/)
