**AWS Interconnect - multicloud** 是 AWS 与 Google Cloud（后续将支持 Azure 和 OCI）深度合作推出的**全托管三层（Layer 3）云对云连接服务**。它旨在消除传统跨云组网中复杂的物理配置和路由器管理。 [[1](https://aws.amazon.com/interconnect/multicloud/), [2](https://aws.amazon.com/blogs/aws/aws-interconnect-is-now-generally-available-with-a-new-option-to-simplify-last-mile-connectivity/), [3](https://dev.classmethod.jp/en/articles/aws-and-google-cloud-aws-interconnect-multicloud-ga/), [4](https://www.youtube.com/watch?v=yVpR2jtffuY)]

与传统方案（DX VIF + GCP CCI）的区别

传统方案通常需要通过第三方服务商（如 [Megaport](https://www.megaport.com/blog/multicloud-connectivity-complete-guide/)  或 [Equinix Fabric](https://www.linkedin.com/posts/bobsills_what-is-aws-interconnect-activity-7453422790063616000-LRKe) ）手动拉线、配置 BGP 邻居，而新方案将其转化为 **API 驱动的极简体验**。 [[1](https://aws.amazon.com/blogs/networking-and-content-delivery/build-resilient-and-scalable-multicloud-connectivity-architectures-with-aws-interconnect-multicloud/), [2](https://www.linkedin.com/posts/bobsills_what-is-aws-interconnect-activity-7453422790063616000-LRKe), [3](https://aws.amazon.com/blogs/networking-and-content-delivery/build-resilient-and-scalable-multicloud-connectivity-architectures-with-aws-interconnect-multicloud/)]

|特性 [[1](https://www.youtube.com/watch?v=yVpR2jtffuY), [2](https://dev.classmethod.jp/en/articles/aws-and-google-cloud-aws-interconnect-multicloud-ga/), [3](https://www.infoq.com/news/2026/04/aws-interconnect-multicloud-ga/), [4](https://www.youtube.com/watch?v=yfxS9Lizu5M&t=32), [5](https://docs.cloud.google.com/network-connectivity/docs/interconnect/concepts/partner-cci-for-aws-overview), [6](https://blogs.juniper.net/en-us/enterprise-cloud-and-transformation/interconnecting-the-enterprise-multicloud-aws-direct-connect), [7](https://aws.amazon.com/blogs/networking-and-content-delivery/aws-and-google-cloud-collaborate-to-simplify-multicloud-networking/), [8](https://aws.amazon.com/blogs/networking-and-content-delivery/build-resilient-and-scalable-multicloud-connectivity-architectures-with-aws-interconnect-multicloud/), [9](https://www.sdxcentral.com/news/aws-and-google-cloud-unite-for-hyperscale-across-cloud/), [10](https://www.linkedin.com/posts/bobsills_what-is-aws-interconnect-activity-7453422790063616000-LRKe)]|传统方案 (DX VIF + GCP CCI)|AWS Interconnect - multicloud|
|---|---|---|
|**开通时间**|数周至数月（涉及物理链路、多方协调）|**数分钟**（全 API 自动化拨号）|
|**基础设施管理**|**手动**。需维护物理连接、BGP 邻居、对等 IP|**全托管**。无需管理客户路由器、BGP 或 IP 互联地址|
|**冗余设计**|需用户手动设计高可用拓扑|**内置四重冗余**（跨 2+ 物理设施、4 台路由器）|
|**安全性**|需手动配置加密（如 VPN over DX）|**默认开启 MACsec 硬件加密**|
|**费用结构**|包含端口费 + 第三方 Fabric 费 + 流量费|单一带宽费（例如 $3,555/月起，含双方费用）|

核心优势与架构变化

1. **极简连接模型**：你只需在 AWS 侧准备好 **Direct Connect Gateway (DXGW)**，在 Google Cloud 侧准备好 **Cloud Router**，通过 AWS 控制台生成的激活码即可完成对接。
2. **性能保障**：流量直接在 AWS 和 Google Cloud 的骨干网之间交换，不经过公共互联网，延迟更低且稳定。
3. **路由自动化**：路由通过托管服务在双向自动传播，你依然可以利用 [AWS Direct Connect Gateway](https://aws.amazon.com/blogs/networking-and-content-delivery/hybrid-cloud-architectures-using-aws-direct-connect-gateway/)  将连接扩展到多个 VPC 或 [Transit Gateway (TGW)](https://aws.amazon.com/blogs/networking-and-content-delivery/build-resilient-and-scalable-multicloud-connectivity-architectures-with-aws-interconnect-multicloud/) 。 [[1](https://www.infoq.com/news/2026/04/aws-interconnect-multicloud-ga/), [2](https://www.sdxcentral.com/news/aws-and-google-cloud-unite-for-hyperscale-across-cloud/), [3](https://www.youtube.com/watch?v=yVpR2jtffuY), [4](https://aws.amazon.com/directconnect/), [5](https://www.flexential.com/resources/blog/aws-direct-connect-guide), [6](https://aws.amazon.com/blogs/networking-and-content-delivery/build-resilient-and-scalable-multicloud-connectivity-architectures-with-aws-interconnect-multicloud/), [7](https://www.megaport.com/blog/aws-azure-google-cloud-the-big-three-compared/)]

如果你目前已有现成的第三方多云互联环境且流量巨大，**AWS Interconnect** 的固定带宽计费模式在超过一定流量（如约 25TB/月）后可能更具成本优势。 [[1](https://www.linkedin.com/posts/bobsills_what-is-aws-interconnect-activity-7453422790063616000-LRKe)]
