# Design Document: PixelPerfect (Figma-to-Code)

## Overview

**PixelPerfect** is a feature within Nunzi that allows the agent to autonomously translate Figma designs into pixel-perfect React/Tailwind components. Unlike simple "screenshot-to-code" tools, PixelPerfect uses the **Figma API** to extract the exact vector node graph (layout, typography, colors, spacing) and validates the result using visual regression testing in the sandbox.

## Workflow

1.  **User Input:**
    *   User provides a Figma File URL and a specific Frame/Node ID (e.g., "Login Screen").
    *   User specifies the tech stack (e.g., "React + Tailwind + shadcn/ui").

2.  **Design Extraction (Figma API):**
    *   The agent queries `GET /v1/files/:key/nodes?ids=:ids` to get the JSON representation of the design.
    *   It extracts:
        *   **Layout:** Auto-layout properties (flex/grid), padding, gaps.
        *   **Typography:** Font family, size, weight, line height.
        *   **Colors:** Fills, strokes, effects (shadows).
        *   **Assets:** It downloads images/icons as SVGs/PNGs using `GET /v1/images/:key`.

3.  **Code Generation (LLM):**
    *   The agent prompts the LLM with a structured representation of the Figma node graph.
    *   **Prompt:** "Generate a React component for this Login Screen. Use `flex-col` for the main container (gap: 24px). The title is 'Inter', 24px, bold. The button is primary color #3B82F6..."
    *   The LLM generates the component file (e.g., `LoginScreen.tsx`).

4.  **Visual Validation (Sandbox):**
    *   The agent spins up a Storybook or preview server in the sandbox.
    *   It uses **Playwright** to take a screenshot of the rendered component.
    *   It compares the screenshot against the original Figma frame image using pixel-diffing (e.g., `pixelmatch`).

5.  **Self-Correction Loop:**
    *   If the diff > 1% (e.g., button padding is wrong), the agent reads the diff report.
    *   It iterates: "The button padding is too small. Increasing `py-2` to `py-4`."
    *   It re-runs the validation until the design matches perfectly.

## Architecture

```mermaid
graph TD
    User[User: Figma URL] --> Agent
    Agent --> FigmaAPI[Figma API]
    FigmaAPI -->|Node Graph JSON| Agent
    FigmaAPI -->|Assets (SVG/PNG)| Agent
    Agent --> LLM[LLM (Code Gen)]
    LLM -->|React Code| Sandbox[Sandbox Environment]
    Sandbox -->|Render| Browser[Headless Browser]
    Browser -->|Screenshot| DiffEngine[Visual Diff Engine]
    FigmaAPI -->|Original Image| DiffEngine
    DiffEngine -->|Diff Score| Agent
    Agent -->|Loop if > 1%| LLM
```

## Technical Requirements

*   **Figma Access Token:** Required to access the API.
*   **Playwright:** Installed in the sandbox for screenshots.
*   **Pixelmatch:** Library for image comparison.

## Value Proposition
*   **Speed:** Reduces frontend implementation time by 90%.
*   **Accuracy:** Eliminates "pixel pushing" and ensures the implementation matches the design intent exactly.
*   **Maintainability:** Generates clean, semantic code (not just absolute positioning).
