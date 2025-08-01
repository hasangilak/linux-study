# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a comprehensive Linux learning curriculum repository containing 32 planned lessons for experienced developers transitioning to Linux system mastery. The curriculum focuses on deep system understanding rather than basic commands.

## Teaching Methodology: STAR Method

All lessons now use the STAR (Situation, Task, Action, Result) method for teaching Linux concepts through real-world production scenarios. This approach ensures practical, immediately applicable knowledge.

For detailed guidelines on creating or enhancing lessons, refer to `STAR-METHOD-GUIDELINES.md`.

## Repository Structure

The curriculum is organized into 6 main sections with markdown files for each lesson:
- Section 1: System Architecture & Boot Process (Lessons 1-5)
- Section 2: Advanced File Systems & Storage (Lessons 6-11) 
- Section 3: Process Management & Performance (Lessons 12-16)
- Section 4: Networking Deep Dive (Lessons 17-22)
- Section 5: Security & Hardening (Lessons 23-27)
- Section 6: Advanced Topics & Automation (Lessons 28-32)

Currently implemented: 
- Lesson 1: 01-kernel-architecture.md (STAR method applied)
- Lesson 2: 02-systemd-init.md
- Lesson 3: 03-boot-process.md
- Guidelines: STAR-METHOD-GUIDELINES.md

## Lesson Structure (STAR Method)

Each lesson markdown file follows this enhanced structure:
1. **Overview** - Brief introduction to the topic
2. **Learning Approach** - Explanation of STAR method usage
3. **Learning Objectives** - Clear goals for the lesson
4. **Core Concepts** - Theory with diagrams
5. **STAR Scenarios** - Real-world production scenarios for each major topic
6. **Advanced Techniques** - Deeper exploration
7. **STAR Learning Exercises** - Hands-on practice scenarios
8. **Commands Reference** - Quick lookup with scenario-based examples
9. **Further Reading** - Additional resources
10. **Next Lesson** - Navigation to next topic

## STAR Scenario Components

Each scenario must include:
- **Situation**: Real production problem with business impact
- **Task**: Clear, measurable objectives
- **Action**: Step-by-step solution with explanations
- **Result**: Quantifiable improvements and lessons learned

## Working with This Repository

When creating new lessons or modifying existing ones:
- Follow the STAR method guidelines in `STAR-METHOD-GUIDELINES.md`
- Include 5-7 STAR scenarios per major topic
- Provide quantifiable results (performance gains, cost savings, etc.)
- Test all commands on Ubuntu 24 LTS
- Focus on production-ready solutions
- Include emergency response, optimization, and troubleshooting scenarios
- Each exercise should build real-world skills

## Quality Standards

All scenarios must:
- Be based on realistic production situations
- Include measurable success criteria
- Provide step-by-step troubleshooting approaches
- Show both investigation and resolution phases
- Include configuration persistence where applicable

## Target Audience

The curriculum assumes:
- 4+ years of Linux usage experience
- Software development background
- Familiarity with command line basics
- Ubuntu 24 LTS as the learning environment
- Interest in production system administration
- Need for practical, immediately applicable skills