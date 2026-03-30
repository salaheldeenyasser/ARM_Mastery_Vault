# ARM Mastery Vault

A structured, project-driven knowledge vault for mastering ARM and embedded systems, from foundations to professional-level topics.

This vault is organized as a progressive path: each stage builds on the previous one, with maps of content (MOCs), hands-on projects, and interview-focused review.

## What This Vault Covers

- ARM architecture fundamentals (profiles, programmer model, registers)
- ARM assembly and calling conventions
- Memory systems (layout, alignment, endianness, MPU/MMU)
- Cortex-M internals (exceptions, NVIC, SysTick, low power)
- Bare-metal peripheral programming (GPIO, UART, SPI, I2C, timers)
- ARM and C integration (startup, linker scripts, vector table)
- RTOS concepts and FreeRTOS on ARM
- Performance optimization and advanced topics (TrustZone, NEON, Linux)
- Debugging/tooling workflows (GDB, OpenOCD, JTAG/SWD, hard faults)
- Real projects and interview preparation

## Vault Structure

The vault follows a stage-based layout:

- `00_Roadmap/` - timeline and study progression
- `01_Foundations/` - CPU and embedded basics
- `02_ARM_Architecture/` - ARM core concepts
- `03_ARM_Assembly/` - assembly programming
- `04_Memory_System/` - memory behavior and organization
- `05_Cortex_M_Deep_Dive/` - Cortex-M internals
- `06_Peripherals_BareMetal/` - driver-level hardware interaction
- `07_ARM_C_Integration/` - boot, startup, linker, sections
- `08_RTOS_on_ARM/` - task scheduling, sync, context switching
- `09_Performance_Optimization/` - timing, cache, compiler and ASM tuning
- `10_Advanced_Topics/` - multi-core, TrustZone, Linux, NEON
- `11_Debugging_Tooling/` - practical debugging stack
- `12_Projects/` - implementation-focused projects
- `13_Interview_Prep/` - topic-wise Q&A and scenarios
- `14_Quick_Reference/` - fast lookup notes and cheat sheets

## Start Here

- Vault home: [[_Dashboard/🏠 ARM Mastery Vault Home]]
- Learning roadmap: [[00_Roadmap/Learning_Roadmap]]

If you are new to ARM:

1. [[01_Foundations/MOC_Foundations]]
2. [[02_ARM_Architecture/MOC_ARM_Architecture]]
3. [[03_ARM_Assembly/MOC_Assembly]]

If you already know embedded basics:

1. [[02_ARM_Architecture/MOC_ARM_Architecture]]
2. [[05_Cortex_M_Deep_Dive/MOC_CortexM]]
3. [[06_Peripherals_BareMetal/MOC_Peripherals]]

If your goal is interview prep:

1. [[13_Interview_Prep/MOC_Interview]]
2. [[11_Debugging_Tooling/MOC_Debugging]]
3. [[14_Quick_Reference/MOC_Reference]]

## How To Use This Vault Effectively

1. Follow each stage in order unless you already have prior depth.
2. Begin each stage from its MOC note.
3. Build notes through links instead of isolated pages.
4. Pair theory with projects in `12_Projects/`.
5. Revisit quick references and interview notes regularly.

## Suggested Workflow (Weekly)

1. Study 3-5 concept notes.
2. Summarize key points in your own words.
3. Implement one small experiment or driver task.
4. Debug the result with GDB/OpenOCD.
5. Capture takeaways in project or topic notes.

## Tooling

This vault is designed primarily for Obsidian (wikilinks, MOCs, graph navigation), but the folder structure and Markdown files also work in standard editors and on GitHub.

## Status Tracking

Use checkboxes in roadmap and MOC notes to track completion per stage. A practical milestone is to complete each stage with at least one code artifact or debugging walkthrough.

## License

Personal learning vault template/content for educational use.
