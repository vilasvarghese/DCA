#Why Docker swarm mode is using a consensus algorithm?
	* Manager nodes: Nodes in charge of managing and scheduling tasks
	* Ensure manager nodes are storing the same consistent state.
	* In case of a failure
		* Any Manager node can pick up the tasks and continue.

#Systems using consensus algorithms 
	* Ensure cluster state is consistent by requiring a majority of nodes to agree on values.
	* Raft tolerates up to (N-1)/2 failures 
	* Requires a majority or quorum of (N/2)+1 members to agree on values proposed
		*e.g. Cluster of 5 Managers running Raft
			* 3 nodes are unavailable
			* system cannot process any more requests to schedule additional tasks. 
			* The existing tasks keep running 
			* Scheduler cannot rebalance tasks to cope with failures as quorum is not formed.

The implementation of the consensus algorithm in swarm mode means it features the properties inherent to distributed systems:

agreement on values in a fault tolerant system. (Refer to FLP impossibility theorem and the Raft Consensus Algorithm paper)
mutual exclusion through the leader election process
cluster membership management
globally consistent object sequencing and CAS (compare-and-swap) primitives
