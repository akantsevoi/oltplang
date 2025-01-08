
| number | function name                         | note about the function                                                                                  |
| ------ | ------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| 1      | consensus                             | makes sure the order of events is identical on each node                                                 |
| 1.1    | leader election                       |                                                                                                          |
| 1.1.1  | alive checks                          | makes sure                                                                                               |
| 1.2    | request's lineriazability             |                                                                                                          |
| 1.2.1  | sequence id generator                 |                                                                                                          |
| 1.3    | consensus confirmation                |                                                                                                          |
|        |                                       |                                                                                                          |
| 2      | healthy cluster                       | "control plan" makes sure the cluster is healthy and cluster's metrics(ex: synchronization lag) are good |
| 2.1    | needed amount of nodes in the cluster |                                                                                                          |
| 2.1.1  | state restoration                     |                                                                                                          |
| 2.2    | secretes management                   |                                                                                                          |
|        |                                       |                                                                                                          |
| 3      | transport                             | anything related to the transport: maintain connections, protocol, etc.                                  |
| 3.1    | transport between nodes in cluster    |                                                                                                          |
| 3.2    | transport between cluster and clients |                                                                                                          |
|        |                                       |                                                                                                          |
| 4      | logic execution                       | business logic execution                                                                                 |
| 4.1    |                                       |                                                                                                          |
|        |                                       |                                                                                                          |
| 5      | storage                               | storage on nodes(leader + followers)                                                                     |
| 5.1    | store sequence of operations          |                                                                                                          |
| 5.2    | store object meta info                | hash, version, etc.                                                                                      |
|        |                                       |                                                                                                          |
| 6      | durability                            |                                                                                                          |
| 6.1    | backup                                |                                                                                                          |
| 6.2    | restore                               |                                                                                                          |
| 6.3    | drift checks                          | bit flips, write errors, etc.                                                                            |
|        |                                       |                                                                                                          |
| 7      | gateway                               | client's sidecar                                                                                         |
| 7.1    | sends requests                        | knows where to send, who is the leader                                                                   |
| 7.2    | receives pushes                       | pushes from cluster, server-side-events                                                                  |
| 7.3    | DSL parser                            |                                                                                                          |
