### Inspecting the cluster
1. GET /_cluster/health - cluster_name, status, node counts, relocating status, etc
2. GET /_cat/nodes?v - ipaddress, performance
3. GET /_cat/indices?v - look at indices, primary shards, document count, size etc.

### Index management
1. PUT /movies - add a `movies` index, default t