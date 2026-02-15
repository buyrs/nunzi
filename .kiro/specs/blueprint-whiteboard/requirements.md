# Requirements Document

## Introduction

Project Blueprint is a built-in whiteboard feature for OpenHands that allows users to visually specify system architectures, UI layouts, and component relationships using an Excalidraw-based drawing canvas. The whiteboard data is analyzed by the AI agent to scaffold project structures, Docker configurations, and code components that match the user's visual design. This bridges the gap between architectural thinking and implementation, accelerating system design and prototyping.

## Glossary

- **Whiteboard_Canvas**: The Excalidraw-based interactive drawing surface embedded in the OpenHands frontend where users create visual diagrams.
- **Blueprint**: A saved whiteboard diagram consisting of Excalidraw vector element data and an exported PNG snapshot, associated with a conversation.
- **Blueprint_Analyzer**: The backend service that processes Blueprint data (vector elements and image) and produces a structured description for the AI agent.
- **Element**: An individual drawing primitive on the Whiteboard_Canvas, such as a rectangle, ellipse, arrow, line, or text label.
- **Scaffold_Prompt**: The structured textual description generated from a Blueprint that is sent to the AI agent as a MessageAction to trigger code generation.
- **Conversation**: An existing OpenHands session between a user and the AI agent.

## Requirements

### Requirement 1: Whiteboard Canvas Integration

**User Story:** As a developer, I want an embedded drawing canvas within the OpenHands UI, so that I can visually sketch architectures and UI layouts without leaving the application.

#### Acceptance Criteria

1. WHEN a user opens the whiteboard view within a Conversation, THE Whiteboard_Canvas SHALL render an interactive Excalidraw drawing surface that supports rectangles, ellipses, arrows, lines, freehand drawing, and text labels.
2. WHEN a user draws, moves, resizes, or deletes an Element on the Whiteboard_Canvas, THE Whiteboard_Canvas SHALL update the in-memory Excalidraw element array in real time.
3. WHEN the Whiteboard_Canvas is empty, THE Whiteboard_Canvas SHALL display placeholder guidance text instructing the user on how to begin drawing.
4. WHEN the Whiteboard_Canvas loads within a Conversation that has an existing Blueprint, THE Whiteboard_Canvas SHALL restore the previously saved Excalidraw element data.

### Requirement 2: Blueprint Persistence

**User Story:** As a developer, I want my whiteboard diagrams saved automatically, so that I can return to a Conversation and continue editing without losing work.

#### Acceptance Criteria

1. WHEN a user modifies the Whiteboard_Canvas and a debounce interval of 2 seconds elapses without further changes, THE system SHALL save the current Excalidraw element array to the backend as a Blueprint associated with the current Conversation.
2. WHEN the backend receives a save request for a Blueprint, THE system SHALL store the Excalidraw element JSON and a PNG snapshot of the canvas.
3. WHEN a save operation fails due to a network error, THE system SHALL display an error notification to the user and retain the unsaved data in memory.
4. WHEN a user navigates back to a Conversation with a saved Blueprint, THE system SHALL retrieve the stored Blueprint and restore the Whiteboard_Canvas to the saved state.
5. WHEN the backend receives a request to retrieve a Blueprint for a Conversation that has no saved Blueprint, THE system SHALL return an empty Blueprint response with a status indicating no data exists.

### Requirement 3: Blueprint Analysis and Structured Description

**User Story:** As a developer, I want the system to interpret my whiteboard drawing and produce a structured description, so that the AI agent can understand my architectural intent.

#### Acceptance Criteria

1. WHEN a user submits a Blueprint for code generation, THE Blueprint_Analyzer SHALL extract labeled nodes (boxes, cylinders, ellipses with text) and connections (arrows, lines between nodes) from the Excalidraw element array.
2. WHEN the Blueprint_Analyzer processes a Blueprint, THE Blueprint_Analyzer SHALL classify each labeled node by its annotation (e.g., "Auth Service (FastAPI)" yields name "Auth Service" and technology "FastAPI").
3. WHEN the Blueprint_Analyzer processes a Blueprint, THE Blueprint_Analyzer SHALL produce a structured JSON description containing a list of nodes (with name, type, technology) and a list of edges (with source node, target node, and optional label).
4. WHEN the Blueprint_Analyzer encounters an Element with no text label, THE Blueprint_Analyzer SHALL assign a default name based on the Element shape and a sequential index (e.g., "Rectangle_1").
5. WHEN the Blueprint_Analyzer encounters an arrow that does not connect two nodes, THE Blueprint_Analyzer SHALL omit that arrow from the edges list and include a warning in the output.
6. THE Blueprint_Analyzer SHALL serialize the structured JSON description to a string and deserialize it back to an equivalent object without data loss (round-trip property).

### Requirement 4: Code Generation via Agent

**User Story:** As a developer, I want the AI agent to scaffold a project based on my whiteboard diagram, so that I can go from visual design to working code quickly.

#### Acceptance Criteria

1. WHEN a user clicks the "Generate Code" button on the Whiteboard_Canvas, THE system SHALL export the Whiteboard_Canvas as a PNG image, run the Blueprint_Analyzer, and send a Scaffold_Prompt containing the structured description and the PNG image to the AI agent as a MessageAction.
2. WHEN the Scaffold_Prompt is sent to the AI agent, THE Scaffold_Prompt SHALL include the structured JSON description, the PNG image URL, and a system instruction directing the agent to scaffold the project.
3. WHEN the AI agent receives a Scaffold_Prompt, THE agent SHALL generate project files including directory structure, configuration files (e.g., docker-compose.yml), and stub code for each identified node.
4. IF the Blueprint_Analyzer produces warnings (e.g., unlabeled nodes, disconnected arrows), THEN THE system SHALL include those warnings in the Scaffold_Prompt so the agent can request clarification from the user.

### Requirement 5: Blueprint API Endpoints

**User Story:** As a frontend developer, I want well-defined API endpoints for saving and loading Blueprints, so that the frontend can persist and retrieve whiteboard data reliably.

#### Acceptance Criteria

1. THE system SHALL expose a PUT endpoint at `/api/conversations/{conversation_id}/blueprint` that accepts Excalidraw element JSON and a PNG image, and stores them as a Blueprint.
2. THE system SHALL expose a GET endpoint at `/api/conversations/{conversation_id}/blueprint` that returns the stored Blueprint data (Excalidraw element JSON and PNG image URL) for the given Conversation.
3. THE system SHALL expose a DELETE endpoint at `/api/conversations/{conversation_id}/blueprint` that removes the stored Blueprint for the given Conversation.
4. WHEN a request is made to any Blueprint endpoint with an invalid or nonexistent Conversation ID, THE system SHALL return an HTTP 404 response with a descriptive error message.
5. WHEN a PUT request is made with malformed Excalidraw element JSON, THE system SHALL return an HTTP 422 response with a validation error message.

### Requirement 6: Whiteboard UI Controls

**User Story:** As a developer, I want clear controls on the whiteboard to generate code and manage my diagram, so that I can easily trigger the code scaffolding workflow.

#### Acceptance Criteria

1. WHEN the Whiteboard_Canvas contains at least one Element, THE system SHALL enable the "Generate Code" button.
2. WHEN the Whiteboard_Canvas contains zero Elements, THE system SHALL disable the "Generate Code" button.
3. WHEN a user clicks the "Generate Code" button, THE system SHALL display a loading indicator until the Scaffold_Prompt has been sent to the agent.
4. WHEN the code generation process completes sending the Scaffold_Prompt, THE system SHALL navigate the user to the Conversation chat view so they can observe the agent working.
5. WHEN a user clicks a "Clear Canvas" button, THE system SHALL remove all Elements from the Whiteboard_Canvas and delete the associated Blueprint from the backend.

### Requirement 7: Whiteboard Navigation and Access

**User Story:** As a developer, I want to access the whiteboard from within a conversation, so that I can switch between chatting with the agent and drawing diagrams seamlessly.

#### Acceptance Criteria

1. WHEN a user is in a Conversation view, THE system SHALL display a tab or toggle that allows switching between the chat view and the Whiteboard_Canvas view.
2. WHEN a user switches from the Whiteboard_Canvas view to the chat view, THE system SHALL preserve the current Whiteboard_Canvas state without data loss.
3. WHEN a user switches from the chat view to the Whiteboard_Canvas view, THE system SHALL restore the Whiteboard_Canvas to the last known state for that Conversation.
