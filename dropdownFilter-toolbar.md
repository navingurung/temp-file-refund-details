```mermaid
flowchart LR
    A[System Status (English)] --> B[Translate to Japanese Label]
    B --> C[Display in Table]

    D[User selects a Japanese status from dropdown] --> E[System checks matching statuses]
    A --> E

    E --> F{Match?}
    F -->|Yes| G[Show row in table]
    F -->|No| H[Hide row]
```
