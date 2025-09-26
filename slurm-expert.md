

Approach
- determine short or long jobs and send to approariate queues 
- for gpu required jobs submit jobs to gpu_short or compoc_gpu
- non-gpu jobs should be send to cpu_short and compoc_cpu
- send short duration jobs to cpu_short or gpu_short partitions
- select free nodes to submit jobs to 