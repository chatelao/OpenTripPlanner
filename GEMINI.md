# Goal
Portiere ausschliesslich den "--load" Teil der Funktionalitäten von Java zusätzlich auf PHP, damit das extern vorcompilierte "graph.obj" via HTTP/HTTPS bereitgestellt werden kann.

# Structure
- `CONCEPT.md`: The overall structure of the product, including Business & Use Cases as well as the High-Level Architecture.
- `DESIGN.md`: The detailed design of the solution, including the architecture, used tech stack for development, production and testing, etc.
- `TOP_ARCHITECTURE.puml`: The top-level component overview showing all major modules and the dataflows between them.
- `ROADMAP.md`: The list of accomplished and planned steps of the project.
- `TECHNICAL_DEBTS.md`: If you find technical debts, like outdate components, security flaws, old patterns, etc. log them here, but don’t fix them until asked to do so.
- `README.md`: Add a link to the GitHub pages if any are produced / present.
- `/specification/`: External Know-How as datasheet, standards, etc. Should be converted to Markdown if PDF, etc.
- `/src/`: The source code of the project
- `/test/`: All tools, configurations & test cases
- `/build/`: Only temporary place for compilation, may be cached by Github

# `CONCEPT.md` handling
- The `CONCEPT.md` add the business and use cases to the top Goal this file.
- It contains an architecture with top-level functional components and their business interfaces.
- It does not contain all precise implementation choices.
- Every major choice is first drawn out as three alternatives, the best one is chosen and the ohter, discarded ones kept in summary in the last chapter of the concept.

# `DESIGN.md`: 
- The `DESIGN.md` derives all necessary technological choices from `CONCEPT.md`.
- It does contain precise implementation choices.
- Every major choice is first drawn out as three alternatives, the best one is chosen and the ohter, discarded ones kept in summary in the last chapter of the concept.
- It contains a detailed architecture of all components and their technical interfaces.
- Included the TOP_ARCHITECTURE.puml` as dynamic-rendering image as soon as available.

## REST-API
- If the API is used, always regenerate the client SDKs and the server stubs from the original interface definitions
- If there are OpenAPI definitions, render them with redocly to the GH page directory /apis.
- If there are othe API definitions, render them with suitable tools to the GH page directory /apis too.
- Verify and add descriptions to the API definitions until they are useable for any product manager / developer using them later.
- If there are REST-APIs, describe them in `api/openapi.yaml`
- If there are SOAP-APIs, describe them in `api/api.wsdl`
- Keep the api definitions up-to-date with every system change

## Command-Line Interface (CLI)
- Every option should be available as short and long form

## Database
- If SQL statements are required, use the oldest, most common standard dialect available to improve cross-plattform compatibility.
- Draw database schematics as plantUML entities with crowfoot notation.

## GUI / UI
- Create screenshots of each UI step tested and store it as asset of the Action Workflow for review



# `ROADMAP.md` handling
- The `ROADMAP.md` is the final plan to implement the `CONCEPT.md` and `DESIGN.md` to achive the top goal
- Define the steps in a way to allow for parallelization by defining interfaces only first and implementing functions later.
- The `ROADMAP.md` file is structured into several key sections:
  - **Progress Overview**: A table summarizing Phases, Descriptions, and Status (using ✅ for completed, 🚧 for in-progress, ⏳ for planned).
  - **Goals**: A high-level list of project objectives with status emojis.
  - **Phases**: Detailed chapters for each project phase.
- The Tasks, and Subtasks if necessary, have checkboxes to show the progress.
- Every task to be implemented has to be modest, feasible and reasonable.
  - If no such task is available, then break down a bigger steps to modest ones without implementing anything, just changing the `ROADMAP.md`.
- Status Emojis:
  - ✅: Completed
  - 🚧: In Progress
  - ⏳: Planned / To Do
- The progress is updated with every increment.
- The finished tasks are linked to the corresponding issue and timestamped at the end of the line.

# HOWTO
- Keep `src/install.sh` to install all tools to build the application (test only tools, see below)
- Used “ReadTheDocs.org” (RTD) for documentation publishing from main branch

# Testing Locally & with Github Action Workflow (GH Action WF)
- Setup the empty CI/CD pipeline before coding anything
- Write CI/CD test independent as `test` script of the Github action workflows
- Create screenshots of each UI step tested and store it as asset of the Action Workflow for review
- Use `test/install.sh` to install test tools.
- Use the Github action workflows to run the tests after commits.
- Before committing fetch all changes from the remote repository and merge the changes
- Run the CI/CD on every commit on every branch
- Add as much caching as possible to the Github action workflows


