# Contribution #943: Optimize receive_msg ioctl to copy only actual received entries

**Contribution Number:** 1
**Student:** David Ajao
**Issue:** (https://github.com/autowarefoundation/agnocast/issues/943)
**Status:** Phase I - Complete

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

[Which parts of the codebase are involved?]

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
'''
if (ret == 0) {
    if (copy_to_user(arg,
                     &receive_msg_args,
                     sizeof(receive_msg_args)))
        return -EFAULT;
}
'''
##### Findings

The entire ioctl_recieve_msg_args union is copied back to the userspace regardless of the value of ret_entr_nums

##### Impact
Given that recieve_msg_core only fills the first ret_entry_nums entries of the 
ret_entry_ids[] and ret_entry_addrs[] (members of the ioctl_recieve_msg_args union) this makes serialization proportional to the buffer size, rather than the amount of data filled.

##### Future Benchmark Results
Before:
X ms

After:
Y ms

Improvement:
Z%
## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
