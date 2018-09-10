# Cluster lifecycle {#concept_fqv_1v2_xdb .concept}

|Status|Description|
|------|-----------|
|inactive|The successfully created cluster does not contain any node.|
|initial|The cluster is applying for corresponding cloud resources.|
|running|The cluster successfully applied for the cloud resources.|
|updating|The cluster is upgrading the Agent.|
|scaling|Change the number of cluster nodes.|
|failed|The cluster application for cloud resources failed.|
|deleting|The cluster is being deleted.|
|delete\_failed|The cluster failed to be deleted.|
|deleted \(invisible to users\)|The cluster is successfully deleted.|

![](images/4752_en-US.png "Cluster status flow")

