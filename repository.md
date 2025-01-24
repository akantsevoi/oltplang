Any DB with adapter that conforms a specific [adapter protocol]:
- base K-V operations: R/W/U/D

The storage will be used mostly as a K-V store.
- [TODO:?] what does it mean for multi-level nested objects?
    - do we want some advanced support for multi-level indexing, update, etc in general?
    - do we still operate them as one object? 
    - or we just simply say: we have key, and we have everything else?

Under the adapter there might be any configuration:
- simple SQLite storage
- complex, distributed, multi-master, bla-bla-bla configuration
- NoSQL/SQL