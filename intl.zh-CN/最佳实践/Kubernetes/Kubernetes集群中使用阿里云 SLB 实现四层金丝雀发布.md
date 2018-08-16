# Kubernetes集群中使用阿里云 SLB 实现四层金丝雀发布 {#concept_tcr_drq_zdb .concept}

在 Kubernetes 集群中，对于通过 tcp/udp 访问的服务来说，七层的 ingress 不能很好地实现灰度发布的需求。这里我们就来介绍一下如何使用 SLB 来进行四层的金丝雀发布。

## 前提条件 {#section_q2n_wkz_zdb .section}

-   您已成功创建一个 Kubernetes 集群，参见[创建Kubernetes集群](../../../../intl.zh-CN/用户指南/Kubernetes 集群/集群管理/创建Kubernetes集群.md#)。
-   SSH 连接到 Master 节点，参见[SSH 访问 Kubernetes 集群](../../../../intl.zh-CN/用户指南/Kubernetes 集群/集群管理/SSH访问Kubernetes集群.md#)。

## 步骤1 部署旧版本服务 {#section_zct_zkz_zdb .section}

1.  登录[容器服务管理控制台](https://cs.console.aliyun.com)。
2.  在 Kubernetes 菜单下，单击左侧导航栏中的**应用** \> **部署**，进入部署列表页面。
3.  单击页面右上角的**使用模板创建**。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/14271/15344047355705_zh-CN.png)

4.  选择所需的集群，命名空间，选择样例模板或自定义，然后单击**创建**。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/14271/15344047355706_zh-CN.png)

    本例是一个 nginx 的编排，通过 SLB 对外暴露的服务。

    ```
    apiVersion: extensions/v1beta1
     kind: Deployment
     metadata:
       labels:
         run: old-nginx
       name: old-nginx
     spec:
       replicas: 1
       selector:
         matchLabels:
           run: old-nginx
       template:
         metadata:
           labels:
             run: old-nginx
             app: nginx
         spec:
           containers:
           - image: registry.cn-hangzhou.aliyuncs.com/xianlu/old-nginx
             imagePullPolicy: Always
             name: old-nginx
             ports:
             - containerPort: 80
               protocol: TCP
           restartPolicy: Always
     ---
     apiVersion: v1
     kind: Service
     metadata:
       labels:
         run: nginx
       name: nginx
     spec:
       ports:
       - port: 80
         protocol: TCP
         targetPort: 80
       selector:
         app: nginx
       sessionAffinity: None
       type: LoadBalancer                       ##通过阿里云SLB对外暴露服务
    ```

5.  单击左侧导航栏中的**应用** \> **部署** ，以及**应用** \> **服务**，进行查看。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/14271/15344047355707_zh-CN.png)

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/14271/15344047355708_zh-CN.png)

6.  您可单击服务右侧的外部端点，进入 nginx 默认欢迎页面，本示例中的 nginx 欢迎页面会显示 **old**，表示当前访问的服务对应后端 old-nginx 容器。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/14271/15344047355711_zh-CN.png)

    为了方便展示多次发布的情况，建议登录 Master 节点，后面都将通过 `curl`命令查看部署的效果。

    ```
    # bash  
    # for x in {1..10} ; do curl EXTERNAL-IP; done                    ##EXTERNAL-IP即是服务的外部端点
    old
    old
    old
    old
    old
    old
    old
    old
    old
    old
    ```


## 步骤2 上线新版本 deployment {#section_cgz_hmz_zdb .section}

1.  登录[容器服务管理控制台](https://cs.console.aliyun.com)。
2.  在 Kubernetes 菜单下，单击左侧导航栏中的**应用** \> **部署**，进入部署列表页面。
3.  单击页面右上角的**使用模板创建**。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/14271/15344047355705_zh-CN.png)

4.  选择所需的集群，命名空间，选择样例模板或自定义，然后单击**创建**。

    本示例是一个 nginx 的新版本的 deployment，其中的 label，也是含有 `app:nginx` 标签。这个标签就是为了使用与旧版本 deployment 相同的 nginx 服务，这样就可以将对应的流量导入进来。

    本示例的编排模板如下。

    ```
    apiVersion: extensions/v1beta1
     kind: Deployment
     metadata:
       labels:
         run: new-nginx
       name: new-nginx
     spec:
       replicas: 1
       selector:
         matchLabels:
           run: new-nginx
       template:
         metadata:
           labels:
             run: new-nginx
             app: nginx
         spec:
           containers:
           - image: registry.cn-hangzhou.aliyuncs.com/xianlu/new-nginx
             imagePullPolicy: Always
             name: new-nginx
             ports:
             - containerPort: 80
               protocol: TCP
           restartPolicy: Always
    ```

5.  单击左侧导航栏中的**部署**，进入部署列表，您可看到名为 new-nginx 的 deployment。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/14271/15344047355712_zh-CN.png)

6.  登录 Master 节点，执行 curl 命名，查看服务访问的情况。

    ```
    # bash  
    # for x in {1..10} ; do curl EXTERNAL-IP; done                    ##EXTERNAL-IP即是服务的外部端点
    new
    new
    new
    old
    new
    old
    new
    new
    old
    old
    ```

    可看到五次访问到旧的服务，五次访问到新的服务。主要原因是 service 对于流量请求是平均的负载均衡策略，而且新老 deployment 均为一个 pod，因此他们的流量百分比为 1：1 。


## 步骤3 调整流量权重 {#section_myf_wkz_zdb .section}

基于 SLB 的金丝雀发布需要通过调整后端的 pod 数量来调整对应的权重。比如我们这里希望新的服务权重更大一些，假如将新的 pod 数量调整到 4 个。

**Note:** 新旧版本应用同时存在时，某个样本的 curl 命令返回的结果并非严格按照设置的权重，本例中执行 10 次 curl 命令，您可通过观察更大的样本获取接近的效果。

1.  登录[容器服务管理控制台](https://cs.console.aliyun.com)。
2.  在 Kubernetes 菜单下，单击左侧导航栏中的**应用** \> **部署**，进入部署列表页面。
3.  选择所需的集群和命名空间，选择所需的 deployment，单击右侧的**更新**。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/14271/15344047355715_zh-CN.png)

4.  在弹出的对抗框中，设置 pod 的数量，将其调整为 4。

    **Note:** Kubernetes 的 Deployment 资源默认的更新方式就是 rollingUpdate，所以在更新过程中，会保证最小可服务的容器个数，该个数也可以在模板里面调整。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/14271/15344047355716_zh-CN.png)

5.  待部署完成后，登录 Master 节点，执行 curl 命名，查看效果。

    ```
    # bash  
     # for x in {1..10} ; do curl EXTERNAL-IP; done                    ##EXTERNAL-IP即是服务的外部端点
     new
     new
     new
     new
     new
     old
     new
     new
     new
     old
    ```


可以看到，10 个请求里面，有 8 个请求到新的服务，2 个到老的服务。

您可通过动态地调整 pod 的数量来调整新老服务的权重，实现金丝雀发布。

