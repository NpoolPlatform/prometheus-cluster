# Prometheus Cluster
[目录](#目录)
- [功能](#功能)
- [配置说明](#配置说明)
- [go-grpc服务监控集成说明](#go-grpc服务监控集成说明)
- [jenkin配置说明](#jenkin配置说明)
- [相关链接](#相关链接)
-----------
### 功能
- [x] 监控k8s集群
- [x] 监控集群内go-grpc相关api
- [ ] 监控集群内第三方相关服务

### 配置说明
* alertmanager/template下存放的是告警通知的模板```当前包含企业微信模板，多个模板配置可新建configMap并在values.yaml中alertmanager.alertmanagerSpec下添加对应的volumes```
  

* grafana-dashboards用于存放grafana dashboard模板 ```添加新的模板直接新建configMap即可，注意:需要在configMap labels中声明grafana_dashboard: "1"```
  

* kube-prometheus-stacky```hellm相关配置```https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack
  

* prometheus/rule下存放prometheus告警规则```添加新的规则直接新建PrometheusRule即可，建议一个groups一个yaml文件，注意：PrometheusRule labels需要声明release: prometheus```


* 告警通知部门和指定用户可以在values.yaml中alertmanager.config.receivers.to_party和to_user中修改

### go-grpc服务监控集成说明
* 修改viper.yaml文件添加prometheus_port:端口 
* service绑定prometheus_port端口
* 声明ServiceMonitor指定service

### jenkin配置说明
需要添加如下选项参数
* WECHAT_CORP_ID ```企业微信企业ID```
* WECHAT_AGENT_ID ```企业微信应用的ID```
* WECHAT_API_SECRET ```企业微信应用的Secret```
* GRAFANA_PASSWORD ```Grafana admin账号密码```
* STORAGE_CLASS_NAME ```pvc storageClassName```


### 相关链接
* [Prometheus Community Kubernetes Helm Charts](https://github.com/prometheus-community/helm-charts)
* [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack#kube-prometheus-stack)
