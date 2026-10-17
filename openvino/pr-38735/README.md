# OpenVINO #38735 — Fix Mismatched Memory Deallocation in Image Wrapper

Fixed a memory allocation/deallocation mismatch in OpenVINO's non-OpenCV image wrapper. Reproduced the issue using AddressSanitizer, verified the fix, and contributed the change to the official master branch after code review and CI validation.

[Pull Request #38735](https://github.com/openvinotoolkit/openvino/pull/38735) · [Merge Commit](https://github.com/openvinotoolkit/openvino/commit/1aada33547823b80f54d2b85703de759ae9e2e18) · [Updated Source Code](https://github.com/openvinotoolkit/openvino/blob/master/samples/c/common/opencv_c_wrapper/src/opencv_c_wrapper.cpp)

## 1. Overview

| Item | Details |
|---|---|
| Project | [Intel OpenVINO](https://github.com/openvinotoolkit/openvino) |
| Pull Request | [#38735 — Fix mismatched deallocation in non-OpenCV image wrapper](https://github.com/openvinotoolkit/openvino/pull/38735) |
| Status | Merged |
| Merge Date | October 16, 2026 |
| Target Branch | master |
| Modified File | samples/c/common/opencv_c_wrapper/src/opencv_c_wrapper.cpp |
| Change Size | 1 file changed, 3 insertions, 1 deletion |

## 2. Problem and Root Cause

While examining the BMP image handling path in the non-OpenCV implementation, I identified a mismatch between the memory allocation and deallocation methods.

The relevant execution flow was:

```text
readBmpImage()
    |
    | Allocates an image buffer using malloc()
    v
image_read()
    |
    | Stores the buffer pointer in img->mat_data
    v
image_free()
    |
    +-- delete img->mat_data;  // Incorrect
```

`readBmpImage()` allocated the image buffer using C's `malloc()`, while `image_free()` attempted to release the same buffer using C++'s `delete`.

In C and C++, dynamically allocated memory must be released using the corresponding deallocation function:

- `malloc()` → `free()`
- `new` → `delete`
- `new[]` → `delete[]`

Using `delete` on memory allocated by `malloc()` results in **undefined behavior**.

## 3. Reproducing the Issue with AddressSanitizer

To confirm the problem, I prepared a minimal test using AddressSanitizer (ASan).

The test loaded a 2×2, 24-bit BMP image through the non-OpenCV wrapper and released the allocated memory.

Before the fix, ASan reported:

```text
ERROR: AddressSanitizer: alloc-dealloc-mismatch
(malloc vs operator delete)
```

This confirmed that the memory deallocation mismatch was reproducible at runtime rather than being only a potential issue identified through code inspection.

## 4. Implementation

I replaced `delete` with `std::free()` in `image_free()` and added the `<cstdlib>` header.

**Before**

```cpp
int image_free(c_mat_t* img) {
    delete img->mat_data;
    return 0;
}
```

**After**

```cpp
#include <cstdlib>

// ...

int image_free(c_mat_t* img) {
    std::free(img->mat_data);
    return 0;
}
```

Since `readBmpImage()` already allocated the buffer using `malloc()`, I kept the allocation logic unchanged and corrected the corresponding deallocation operation.

This resolved the mismatch with a minimal code change.

### Verification

I reran the same ASan test using the original BMP image.

| Test | Before Fix | After Fix |
|---|---|---|
| Input | 2×2, 24-bit BMP | Same image |
| Deallocation | `delete` | `std::free()` |
| ASan Result | `alloc-dealloc-mismatch` | No ASan errors |
| Execution | ASan error reported | Image loaded successfully, exit code 0 |

The program successfully loaded the image and exited without AddressSanitizer errors after the fix.

## 5. Code Review and CI

After submitting the pull request to the official OpenVINO repository, I worked through the project's code review and CI process.

### Fixing a clang-format Failure

The initial CI run failed the `clang-format` checks.

After examining the CI output, I found that the newly added `<cstdlib>` header did not follow the project's required include ordering.

I corrected the header placement and ran the following checks locally:

```bash
git diff --check

clang-format-18 --style=file --dry-run --Werror \
  samples/c/common/opencv_c_wrapper/src/opencv_c_wrapper.cpp
```

Both checks passed, and I pushed a follow-up commit.

[Formatting Fix Commit](https://github.com/openvinotoolkit/openvino/commit/19500def39f1e5222a802a758308eb09089c74b4)

### Jenkins Checks and Maintainer Communication

During CI validation, several Jenkins checks initially failed.

The Jenkins server was accessible only through an internal network, so I could not inspect the detailed logs.

Rather than assuming the cause of the failures, I left a comment asking the maintainers to investigate and confirm whether any changes were required on my side.

[PR Discussion](https://github.com/openvinotoolkit/openvino/pull/38735#issuecomment-6150027592)

The affected checks subsequently passed, and the PR completed CI validation.

### Approval and Merge

The pull request received approvals from three maintainers.

After passing the required checks, the PR was merged into OpenVINO's official master branch through the merge queue.

- **PR:** [openvinotoolkit/openvino#38735](https://github.com/openvinotoolkit/openvino/pull/38735)
- **Merge Commit:** [`1aada33`](https://github.com/openvinotoolkit/openvino/commit/1aada33547823b80f54d2b85703de759ae9e2e18)
- **Merged:** October 16, 2026 (UTC)

## 6. Lessons Learned

### Tracing Memory Ownership

This issue reinforced the importance of tracing where memory is allocated, how the pointer is passed between functions, and where the memory is eventually released.

Inspecting only the deallocation statement would not have been enough to understand the underlying problem.

### Runtime Verification with AddressSanitizer

I used ASan to reproduce a memory management issue and compare the behavior before and after the fix.

This provided practical experience using a runtime diagnostic tool to validate a bug fix rather than relying on code inspection alone.

### Working with an Upstream Project

Even a small change needs to follow a project's coding standards and pass its automated checks.

I also learned to distinguish between issues I could investigate locally and CI failures that required maintainer assistance.

Although the final code change was small, the contribution involved the complete process of identifying a bug, reproducing it, implementing a fix, validating the result, responding to CI feedback, and getting the change merged upstream.

## 7. AI Assistance

AI tools assisted with root cause analysis, preparation of a minimal ASan reproducer, and review of the proposed change.

I personally inspected the allocation and deallocation paths, compiled and tested the affected code with ASan, verified the behavior before and after the fix, and reviewed the final Git diff.

AI assistance was also disclosed in the [original pull request](https://github.com/openvinotoolkit/openvino/pull/38735).