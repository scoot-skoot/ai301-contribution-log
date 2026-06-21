# Contribution #943: Optimize receive_msg ioctl to copy only actual received entries

**Contribution Number:** 1
**Student:** David Ajao
**Issue:** (https://github.com/autowarefoundation/agnocast/issues/943)
**Status:** Phase IV - In-progress

---

## Why I Chose This Issue

I really enjoy learning a lot about low level systems theory and have been looking for good opportunities to put the knowledge into practice. As such, the issue being a well-defined performance engineer problem makes it one of the best issues, I could start with. Through this project, I want to gain experience working with larger, unfamiliar codebases and get really quick at parsing code. I also hope to get a feel for some of the differences between the theory and application of low-level software. I am also looking forward to branching out of the C++ STL.


## Understanding the Issue

### Problem Description

At a very high level, the goal of this contribution is to optimize the IO Control data transfer between the kernel and the user. Currently, the implementation transfers the full buffer from the kernel to the user, even when there is only n pieces of data (where n < buffer size). This is i0efficient as we end up transferring unused array capacity, which artificially constraints the programs efficiency.

### Expected Behavior

Ideally, rather than copying the full buffer every time, the program should be dynamic and allow for data transfer of variable lengths between the kernel and the user.

### Current Behavior

The implementation currently copies the entire ioctl_recieve_msg_args union back to the userspace, regardless of the value of ret_entry_num (number of filled buffer spots)

### Affected Components

The two main files of interest are
- agnocast_ioctl.c
- agnocast.h
  
The functions/objects of interest are
- recieve_msg_cmd()
- recieve_msg_core()
- union ioctl_recieve_msg_args
- copy_to_user(arg, &recieve_msg_args, sizeof(recieve_msg_args))

## Reproduction Process
This is not so much a bug as a feature implementation, however I believe I can compare latency and runtime differences using the repo's own testing suite or RO2.

### Environment Setup
As this is my first time working with kernels in real codebases, I naively though that I would just clone the repo in my WSL and get going. What I observed however is that WSL is not a great environment for kernel work as many of the packages and depdencies needed for the project do not operate well with WSL's kernel (YMMV).
As this is repo primarily serves as middleware between the kernel and the userspace, I had to set up a virtual machine running Ubuntu.
Other than that small hiccup, the set up and build was actually suprisingly smooth. I had the same depency hiccups that is almost customary when building projects for the first time, but it was far less and a lot easier to debug than JS/Typescript projects so that was awesome.
Overall, after the virtual machine system was setup, from cloning to building was less than 2 hours! :)

### Steps to Reproduce

1. Build Agnocast
2. Load kernel module.
3. Run recieve benchmark/test
4. Observe receieve_msg ioctl behavior
5. Compare about of data copied vs ret_entry_num

### Reproduction Evidence

##### Relevant code path identified

```
if (ret == 0) {
    if (copy_to_user(arg,
                     &receive_msg_args,
                     sizeof(receive_msg_args)))
        return -EFAULT;
}
```

#### Findings

The entire ioctl_recieve_msg_args union is copied back to the userspace regardless of the value of ret_entr_nums

#### Impact
Given that recieve_msg_core only fills the first ret_entry_nums entries of the 
ret_entry_ids[] and ret_entry_addrs[] (members of the ioctl_recieve_msg_args union) this makes serialization proportional to the buffer size, rather than the amount of data filled.

### Analysis

The root cause of this issue was that the kernel module was copying the entire `ioctl_receieve_msg_args` structure back to userspace reagrdless of how much data was actual produced.
Since the union structured contains fixed sized arrays for receieving the stored entries, but only the first `ret_entry_num` enteries are valid, this mean thtat when the system receieved a small number of data, it still copied the full buffer, which was a hamper on further optimization down that route, by increasing the buffer size.

### Proposed Solution

The proposed solution was to preserve the existing structure and behavior while changing how the data was copied.
Instead of copying the entire fixed-size arrays, the goal was to redesign the copy function so that it
- Copied only the valid portion `red_entry_ids`
- Copied only the valid portion of `red_entry_addrs`
And by extension, also copying each scaler field indvidually.

The primary consideration I ahd to keep in mind throughout the entire implementation was preserving the userspace's existing contract, so that the overall beahvior of the program would remain unchanged, and I would not break anything.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** 
The receive message ioctl is performing unnecessary kernel-to-userspace copies by transferring unused array capacity.


**Match:** 
After looking around the codebase, the existing code base already provided a natural boundary for valid data through the `ret_entry_num` variable.
Using AI for research similar patterns of copying only active ranges are common when working with dynamically sized data.

**Plan:** [Step-by-step implementation plan]
1. Modify agnocast_ioctl.c to avoid copying the entire receive structure.
2. Copy scalar values from the kernel structure into userspace.
3. Copy only ret_entry_num entries from `ret_entry_ids` and `ret_entry_addrs`
Run existing kernel and integration tests.

**Implement:** 
(https://github.com/autowarefoundation/agnocast/pull/1396)

**Review:** 
After implementating my solution I checked to ensure that
- ABI compatability was preserved (kernel level contracts intact and plays nice with the precompiled files)
- Userspace behavior remained unchanged.
- Only valid entries were copies
- Existing tests continued to pass

**Evaluate:** 
The change would be evaluated by comparing the amount of copied data before and after the change, along with running the existing Agnocast test suite.

---

## Testing Strategy

### Unit Tests

- [x] Test case 1: Verified the modified code path builds successfully.
- [x] Test case 2: Ran existing kernel module tests.
- [x] Test case 3: Confirmed the receive message behavior remained unchanged.

### Integration Tests (Ran Autoware Tests)

- [x] scripts/test/e2e_test_1to1.bash
- [x] scripts/test/e2e_test_2to2.bash

### Manual Testing

I manually inspected the receieve message flow using two terminals and verified that
`ret_entry_num` still controlled the valid range of data, the userspace continued to recieve the expected data, and that only the valid enteries were transfered over.

---

## Implementation Notes

### Week [1] Progress
The first challenge was understanding the codebase structure and identifying where the kernel-to-userspace transfer happened.

Since this was my first time working directly with kernel-related code, I initially approached the issue from a pure performance perspective: fewer bytes copied should mean better performance.

Through review and discussion, I learned that performance changes need to consider the larger architecture. A small optimization does not matter much if the surrounding system has changed and the operation is no longer on a critical path.

### Code Changes

- **Files modified:**
- agnocast_ioctl.c
- agnocast.h
- **Key commits:**
https://github.com/autowarefoundation/agnocast/pull/1396/changes/c6c39a4ac1fe20eabbb141a6880f0a7c0596ac00
- **Approach decisions:**
The main design decision was to avoid changing the ABI. Instead of modifying the data structures, the optimization only changed how existing data was transferred.

---

## Pull Request

**PR Link:** https://github.com/autowarefoundation/agnocast/pull/1396

**PR Description:** Implemented an optimization for the receive message ioctl by reducing unnecessary kernel-to-userspace copies.

Previously, the full fixed-size receive buffer was copied regardless of the number of valid entries. This change copies only the valid prefix based on ret_entry_num.

**Maintainer Feedback:**
-6/15/26: The mainter explained that while the optimzation was reasonable, since the issue was posted the surrounding architecture had changed so that the optimization no longer made sense.
More specifically, since the message reception and callback execution were now combined, the previous lock contention that inspired the creation of the issue no longer applied, meaning `MAX_RECIEVE_NUM` is now expected to remain small instead of being raised.
Thus, the mainters preferred to keep the simplier implementation rather than bring on the added complexity for menial gains.

- 6/17/26: Moved onto a new issue in Agnocast!

**Status:** 
Not Merged (Issue Closed)
---

## Learnings & Reflections
### Technical Skills Gained
Through working on this contribution I learned a lot more about
- Kernel/userspace boundaries
- ioctl communication patterns
- ABI compatability
- Kernel memory cppying
- Performance Engineering

I also became a lot more confortable locating the important pieces of informationa dn tracing execution through larger repositories.

### Challenges Overcome

It was super daunting working with such a large codebase with so many interconnected parts. II learned however, that it is much simplier if you narrow your focus and only worry about what is connected with what.
I also found it super useful to work with a Google doc and sort of construct a map of what does what.

### What I'd Do Differently Next Time

Honestly, this was a bit unlucky so I can't say I would change too much for the next time around. I might move quicker, because I spent a lot of time polishing and double and triple checking instead of just getting out there and taking a swing.

---

## Resources Used
- https://github.com/autowarefoundation/agnocast/tree/main/docs
- (https://github.com/autowarefoundation/agnocast/blob/main/docs/shared_memory.md)
- (https://docs.kernel.org/driver-api/ioctl.html)
