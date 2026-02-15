# Requirements Document

## Introduction

PixelPerfect is a Figma-to-Code agent for the Nunzi platform (built on OpenHands). It connects to the Figma API, extracts design specifications (layout, colors, typography, spacing, components, auto-layout properties), and generates production-ready frontend code that matches the original design. PixelPerfect creates pull requests with generated React components (with CSS/Tailwind styling), design tokens, and visual regression test snapshots. When a Figma design is updated, PixelPerfect detects the changes and opens follow-up PRs to keep code in sync.

## Glossary

- **PixelPerfect_Agent**: The autonomous agent that orchestrates the full Figma-to-code pipeline: design extraction, code generation, PR creation, and change detection.
- **Figma_Connector**: The component responsible for authenticating with the Figma API and retrieving design file data.
- **Design_Tree**: The hierarchical structure extracted from a Figma file, containing frames, components, component sets, styles, auto-layout properties, and constraints.
- **Design_Node**: A single element in the Design_Tree (frame, group, component, text, vector, rectangle, etc.) with its properties.
- **Design_Token**: A named value extracted from Figma styles representing a reusable design decision (color, typography, spacing, border radius, shadow).
- **Token_Set**: A collection of Design_Token objects extracted from a Figma file.
- **Component_Mapper**: The component that maps Figma design nodes to React component structures, determining hierarchy, props, and composition.
- **Component_Spec**: A structured description of a single React component to be generated, including its name, props, children, styles, and accessibility attributes.
- **Component_Spec_Set**: A collection of Component_Spec objects representing all components to generate for a design.
- **Code_Generator**: The component that transforms Component_Spec objects into production-ready React/TypeScript code with Tailwind CSS or CSS module styling.
- **Generated_File**: A single output file produced by the Code_Generator (React component, CSS module, design token file, test snapshot, or barrel export).
- **Generated_File_Set**: The complete collection of Generated_File objects for a single generation run.
- **PR_Creator**: The component that creates a pull request containing the generated code, using the existing OpenHands GitHub infrastructure.
- **Change_Detector**: The component that compares a current Figma Design_Tree against a previously stored Design_Tree to identify what changed.
- **Change_Set**: A structured description of differences between two Design_Tree versions, including added, removed, and modified nodes.
- **PixelPerfect_Config**: A repository-level configuration file (`.pixelperfect.yml`) that controls PixelPerfect behavior, including target framework, styling approach, output directory, and Figma file mappings.
- **Snapshot_Generator**: The component that produces visual regression test snapshots for generated components.

## Requirements

### Requirement 1: Figma Design Input and Authentication

**User Story:** As a developer, I want to provide a Figma file URL or design token so that PixelPerfect can access my design and begin code generation.

#### Acceptance Criteria

1. WHEN a user provides a Figma file URL, THE Figma_Connector SHALL parse the URL to extract the file key and optionally a node ID.
2. WHEN a user provides a Figma personal access token or OAuth token, THE Figma_Connector SHALL authenticate with the Figma API using that token.
3. IF the Figma API returns an authentication error, THEN THE Figma_Connector SHALL return a descriptive error message indicating invalid or expired credentials.
4. IF the Figma API is unreachable or returns a server error, THEN THE Figma_Connector SHALL retry the request up to 3 times with exponential backoff before returning an error.
5. WHEN authentication succeeds, THE Figma_Connector SHALL retrieve the full file data for the specified file key using the Figma REST API.
6. WHEN a node ID is specified in the URL, THE Figma_Connector SHALL retrieve only the subtree rooted at that node.

### Requirement 2: Design Tree Extraction

**User Story:** As a developer, I want PixelPerfect to extract the complete design structure from my Figma file so that it can understand the layout, components, and styles.

#### Acceptance Criteria

1. WHEN a Figma file is retrieved, THE PixelPerfect_Agent SHALL parse the API response into a Design_Tree containing all frames, components, groups, text nodes, vectors, and rectangles.
2. WHEN parsing the Figma response, THE PixelPerfect_Agent SHALL extract auto-layout properties (direction, spacing, padding, alignment) for each applicable node.
3. WHEN parsing the Figma response, THE PixelPerfect_Agent SHALL extract style properties (fill colors, stroke colors, font family, font size, font weight, line height, letter spacing, border radius, shadows, opacity) for each node.
4. WHEN parsing the Figma response, THE PixelPerfect_Agent SHALL extract constraint and sizing properties (fixed width/height, min/max dimensions, fill container, hug contents) for each node.
5. WHEN a Design_Tree is constructed, THE PixelPerfect_Agent SHALL serialize the Design_Tree to JSON for storage and downstream consumption.
6. FOR ALL valid Design_Tree objects, serializing then deserializing SHALL produce an equivalent Design_Tree object (round-trip property).
7. WHEN a Figma component instance references a main component, THE PixelPerfect_Agent SHALL resolve the reference and link the instance to its main component definition in the Design_Tree.

### Requirement 3: Design Token Extraction

**User Story:** As a developer, I want PixelPerfect to extract design tokens from my Figma styles so that I can maintain a consistent design system in code.

#### Acceptance Criteria

1. WHEN a Figma file contains published styles, THE PixelPerfect_Agent SHALL extract color tokens from fill and stroke styles.
2. WHEN a Figma file contains published styles, THE PixelPerfect_Agent SHALL extract typography tokens (font family, size, weight, line height, letter spacing) from text styles.
3. WHEN a Figma file contains consistent spacing values, THE PixelPerfect_Agent SHALL extract spacing tokens from auto-layout gap and padding values.
4. WHEN a Figma file contains effect styles, THE PixelPerfect_Agent SHALL extract shadow and blur tokens.
5. WHEN a Token_Set is constructed, THE PixelPerfect_Agent SHALL serialize the Token_Set to JSON for storage and downstream consumption.
6. FOR ALL valid Token_Set objects, serializing then deserializing SHALL produce an equivalent Token_Set object (round-trip property).
7. THE PixelPerfect_Agent SHALL generate a design token file in the configured format (CSS custom properties, Tailwind config, or TypeScript constants).
8. THE Token_Set Pretty_Printer SHALL format Token_Set objects back into valid design token files.
9. FOR ALL valid Token_Set objects, generating a token file then parsing it back SHALL produce an equivalent Token_Set object (round-trip property).

### Requirement 4: Component Mapping

**User Story:** As a developer, I want PixelPerfect to intelligently map Figma components to React components so that the generated code has a clean component hierarchy.

#### Acceptance Criteria

1. WHEN a Design_Tree is analyzed, THE Component_Mapper SHALL identify Figma components and component sets and map each to a distinct React component.
2. WHEN a Figma component has variants (via component sets), THE Component_Mapper SHALL map variants to React component props.
3. WHEN a Figma frame contains nested components, THE Component_Mapper SHALL produce a Component_Spec with the correct parent-child composition hierarchy.
4. WHEN a Figma node uses auto-layout, THE Component_Mapper SHALL map the layout to corresponding CSS flexbox properties (direction, gap, padding, alignment).
5. WHEN a Figma text node is encountered, THE Component_Mapper SHALL extract the text content and map typography styles to CSS properties.
6. WHEN a Figma node has constraints (e.g., fill container, fixed width), THE Component_Mapper SHALL map constraints to responsive CSS sizing properties.
7. WHEN a Component_Spec_Set is constructed, THE Component_Mapper SHALL serialize the Component_Spec_Set to JSON for storage and downstream consumption.
8. FOR ALL valid Component_Spec_Set objects, serializing then deserializing SHALL produce an equivalent Component_Spec_Set object (round-trip property).

### Requirement 5: Code Generation

**User Story:** As a developer, I want PixelPerfect to generate production-ready React components with proper styling so that I can use them directly in my project.

#### Acceptance Criteria

1. WHEN a Component_Spec is provided, THE Code_Generator SHALL produce a React/TypeScript component file with proper imports, props interface, and JSX structure.
2. WHEN the configured styling approach is Tailwind, THE Code_Generator SHALL generate Tailwind CSS utility classes for all style properties.
3. WHEN the configured styling approach is CSS Modules, THE Code_Generator SHALL generate a companion `.module.css` file with scoped class names.
4. WHEN a component has interactive states (hover, focus, active) defined in Figma, THE Code_Generator SHALL generate corresponding CSS pseudo-class or Tailwind variant styles.
5. THE Code_Generator SHALL generate accessible markup by including appropriate ARIA attributes, semantic HTML elements, and alt text for images.
6. WHEN generating components, THE Code_Generator SHALL produce a barrel export file (`index.ts`) that re-exports all generated components.
7. WHEN generating components, THE Code_Generator SHALL produce a design token file that exports all extracted tokens in the configured format.
8. THE Code_Generator SHALL generate syntactically valid TypeScript that passes type checking.
9. WHEN a Generated_File_Set is constructed, THE Code_Generator SHALL serialize the Generated_File_Set to JSON for storage and downstream consumption.
10. FOR ALL valid Generated_File_Set objects, serializing then deserializing SHALL produce an equivalent Generated_File_Set object (round-trip property).

### Requirement 6: Pull Request Creation

**User Story:** As a developer, I want PixelPerfect to create a PR with the generated components so that I can review and merge the code through my normal workflow.

#### Acceptance Criteria

1. WHEN code generation is complete, THE PR_Creator SHALL create a new branch named `pixelperfect/{figma-file-key}/{timestamp}`.
2. WHEN creating a PR, THE PR_Creator SHALL commit all Generated_File objects to the configured output directory on the new branch.
3. WHEN creating a PR, THE PR_Creator SHALL generate a descriptive PR title including the Figma file name and number of components generated.
4. WHEN creating a PR, THE PR_Creator SHALL generate a PR body containing a summary of generated components, design tokens extracted, and a link back to the Figma file.
5. WHEN creating a PR, THE PR_Creator SHALL use the existing OpenHands GitHub infrastructure (`send_pull_request.py` patterns) to create the pull request.
6. IF creating the branch or PR fails, THEN THE PR_Creator SHALL log the error with context and report the failure in the PixelPerfect_Output.

### Requirement 7: Design Change Detection and Sync

**User Story:** As a developer, I want PixelPerfect to detect when my Figma design is updated and open a follow-up PR to sync the code, so that my implementation stays in sync with the design.

#### Acceptance Criteria

1. WHEN a sync is triggered, THE Change_Detector SHALL compare the current Design_Tree against the previously stored Design_Tree.
2. WHEN comparing Design_Trees, THE Change_Detector SHALL identify added nodes, removed nodes, and modified nodes (property changes).
3. WHEN a Change_Set is produced, THE Change_Detector SHALL serialize the Change_Set to JSON for storage and downstream consumption.
4. FOR ALL valid Change_Set objects, serializing then deserializing SHALL produce an equivalent Change_Set object (round-trip property).
5. WHEN changes are detected, THE PixelPerfect_Agent SHALL regenerate only the affected components and create a follow-up PR with the updates.
6. WHEN creating a follow-up PR, THE PR_Creator SHALL include a summary of what changed in the Figma design in the PR body.
7. IF no changes are detected between the current and stored Design_Tree, THEN THE PixelPerfect_Agent SHALL log that the design is up to date and take no further action.

### Requirement 8: Configuration and Customization

**User Story:** As a developer, I want to configure PixelPerfect's behavior so that the generated code matches my project's conventions and structure.

#### Acceptance Criteria

1. THE PixelPerfect_Config SHALL support a `figma_token` field for the Figma API authentication token.
2. THE PixelPerfect_Config SHALL support a `files` field that maps Figma file URLs to output directories.
3. THE PixelPerfect_Config SHALL support a `framework` field specifying the target framework (default: `react`).
4. THE PixelPerfect_Config SHALL support a `styling` field specifying the styling approach: `tailwind` or `css-modules` (default: `tailwind`).
5. THE PixelPerfect_Config SHALL support a `typescript` field enabling or disabling TypeScript output (default: `true`).
6. THE PixelPerfect_Config SHALL support a `token_format` field specifying the design token output format: `css`, `tailwind`, or `typescript` (default: `tailwind`).
7. THE PixelPerfect_Config SHALL support a `component_prefix` field for prefixing generated component names (default: empty).
8. THE PixelPerfect_Config SHALL support an `ignore_nodes` field specifying Figma node names or IDs to skip during extraction.
9. WHEN a `.pixelperfect.yml` file is present in the repository root, THE PixelPerfect_Agent SHALL load and apply the configuration.
10. IF no `.pixelperfect.yml` file exists, THEN THE PixelPerfect_Agent SHALL use default configuration values.

### Requirement 9: Visual Regression Testing

**User Story:** As a developer, I want PixelPerfect to generate visual regression test snapshots so that I can detect unintended visual changes in future updates.

#### Acceptance Criteria

1. WHEN components are generated, THE Snapshot_Generator SHALL produce a Storybook story file for each generated component.
2. WHEN a Storybook story is generated, THE Snapshot_Generator SHALL include all component variants as separate stories.
3. WHEN a follow-up sync PR is created, THE Snapshot_Generator SHALL update the snapshot files for affected components.
4. IF the PixelPerfect_Config disables snapshot generation, THEN THE Snapshot_Generator SHALL skip snapshot creation.

### Requirement 10: Error Handling and Output

**User Story:** As a developer, I want PixelPerfect to handle errors gracefully and provide clear output so that I can understand what happened during code generation.

#### Acceptance Criteria

1. WHEN any component encounters an unrecoverable error, THE PixelPerfect_Agent SHALL log the error with full context (component name, input data summary, error message) and continue processing remaining components.
2. WHEN the generation run completes (success or partial failure), THE PixelPerfect_Agent SHALL produce a structured PixelPerfect_Output JSON object summarizing the run: Figma file processed, components generated, tokens extracted, PR created, and any errors.
3. WHEN the PixelPerfect_Agent produces a PixelPerfect_Output, THE PixelPerfect_Agent SHALL write the output to the configured output directory as `pixelperfect_output.json`.
4. FOR ALL valid PixelPerfect_Output objects, serializing then deserializing SHALL produce an equivalent PixelPerfect_Output object (round-trip property).
