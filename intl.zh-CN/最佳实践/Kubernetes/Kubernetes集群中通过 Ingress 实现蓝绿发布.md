# Kubernetes集群中通过 Ingress 实现蓝绿发布 {#concept_pvj_zqq_zdb .concept}

在发布应用时，经常需要先上线一个新版本，用较小的流量去测试一下该新版本的可用性。Kubernetes 的 ingress resource 并没有实现流量控制与切分的功能，导致针对同一个域名下的路径，只能有一个 service 来进行服务，这样对于灰度发布十分不利。本文介绍如何利用阿里云容器服务的路由功能，来实现蓝绿发布，轻松实现流量切分。

## 前提条件 {#section_cbh_53z_zdb .section}

-   您已成功创建一个 Kubernetes 集群，参见[创建Kubernetes集群](../../../../intl.zh-CN/用户指南/Kubernetes 集群/集群管理/创建Kubernetes集群.md#)。
-   SSH 连接到 Master 节点，参见[SSH 访问 Kubernetes 集群](../../../../intl.zh-CN/用户指南/Kubernetes 集群/集群管理/SSH访问Kubernetes集群.md#)。

## 步骤1 创建应用 {#section_oz4_v3z_zdb .section}

1.  登录[容器服务管理控制台](https://cs.console.aliyun.com)。
2.  在 Kubernetes 菜单下，单击左侧导航栏中的**应用** \> **部署**，进入部署列表页面。
3.  单击页面右上角的**使用模板创建**。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/14270/15344014475686_zh-CN.png)

4.  选择所需的集群，命名空间，选择样例模板或自定义，然后单击**创建**。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/14270/15344014475687_zh-CN.png)

    本例是一个 nginx 应用，包含一个 deployment、 service 以及 ingress。deployment 通过 NodePort 对外暴露端口，并且有一个 ingress 正在对外提供服务。编排模板如下。

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
         run: old-nginx
       name: old-nginx
     spec:
       ports:
       - port: 80
         protocol: TCP
         targetPort: 80
       selector:
         run: old-nginx
       sessionAffinity: None
       type: NodePort
     ---
     apiVersion: extensions/v1beta1
     kind: Ingress
     metadata:
       name: echo
     spec:
       backend:
         serviceName: default-http-backend
         servicePort: 80
       rules:
       - host: mini-echo.io                           ##虚拟主机名称
         http:
           paths:
           - path: /
             backend:
               serviceName: old-nginx
               servicePort: 80
    ```

5.  创建成功后，单击左侧导航栏中的**应用** \> **路由**。

    您可看到虚拟主机名称指向 old-nginx。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/14270/15344014475689_zh-CN.png)

6.  登录 Master 节点，执行 curl 命令，查看路由的访问情况。

    ```
    # curl -H "Host: mini-echo.io" http://101.37.224.229                ##即 Ingress 的访问地址
    old
    ```


## 步骤2 创建新版本的 deployment 和 service {#section_ezh_5jz_zdb .section}

1.  登录[容器服务管理控制台](https://cs.console.aliyun.com)。
2.  在 Kubernetes 菜单下，单击左侧导航栏中的**应用** \> **部署**，进入部署列表页面。
3.  单击页面右上角的**使用模板创建**。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/14270/15344014475686_zh-CN.png)

4.  选择所需的集群，命名空间，选择样例模板或自定义，然后单击**创建**。

    新版本的 deployment 和service 的编排如下。

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
         spec:
           containers:
           - image: registry.cn-hangzhou.aliyuncs.com/xianlu/new-nginx
             imagePullPolicy: Always
             name: new-nginx
             ports:
             - containerPort: 80
               protocol: TCP
           restartPolicy: Always
     ---
     apiVersion: v1
     kind: Service
     metadata:
       labels:
         run: new-nginx
       name: new-nginx
     spec:
       ports:
       - port: 80
         protocol: TCP
         targetPort: 80
       selector:
         run: new-nginx
       sessionAffinity: None
       type: NodePort
    ```


## 步骤3 修改 Ingress 实现蓝绿发布 {#section_emp_t3z_zdb .section}

1.  登录[容器服务管理控制台](https://cs.console.aliyun.com)。
2.  在 Kubernetes 菜单下，单击左侧导航栏中的**应用** \> **路由**，进入路由列表页面。
3.  选择所需的集群和命名空间，选择前面创建的路由，并单击右侧的**更新**。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/14270/15344014475691_zh-CN.png)

4.  在弹出的对话框中，对 Ingress 进行修改，最后单击**确定**。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/14270/15344014475693_zh-CN.png)

    -   **添加注解（annotations）**： `/`后面为新服务的服务名 new-nginx；50 是一个流量值，代表 50%。标签`ingress.aliyun.weight/new-nginx: "50"`完整含义是将流量的百分之 50 引入到新服务的 pod 里面。
    -   **配置新的 serviceName**：这里是和上面旧版本服务并列，即在相同的 path 下，挂两个不同的 service，分别对应于两个新老应用。
    返回路由列表页面，您可看到该路由新增了一条路由规则，指向新版本的 new-nginx 服务。

    ![](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/14270/15344014475694_zh-CN.png)

5.  登录 Master 节点，执行 curl 命令，查看路由的访问情况。

    ```
    # curl -H "Host: mini-echo.io" http://101.37.224.229               ##即 Ingress 的访问地址
     old
     # curl -H "Host: mini-echo.io" http://101.37.224.229
     new
     # curl -H "Host: mini-echo.io" http://101.37.224.229
     old
     # curl -H "Host: mini-echo.io" http://101.37.224.229
     new
     # curl -H "Host: mini-echo.io" http://101.37.224.229
     old
     # curl -H "Host: mini-echo.io" http://101.37.224.229
     new
    ```

    本例中，执行 6 次 curl 命令，分别得到三次新服务，三次老服务的返回结果，这表明权重设置生效了。


您可通过灵活调整 Ingress 中的注解 `ingress.aliyun.weight/new-nginx: "50"` 的值，来设置蓝绿发布的流量占比。

完成新版本应用测试后，您可将 Ingress 的路由权重设置为 100 将流量完全导向新服务；或者删除 Ingress 中的注解和旧版本服务，实现蓝绿发布。

