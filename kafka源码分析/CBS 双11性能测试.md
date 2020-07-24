### CBS 双11性能测试

#### 服务与接口

| 服务 | 节点数 | 接口 | QPS-avg |  QPS-max | 目标 QPS | 依赖 |
| :--: |:--: |:--: |:--: |:--: |:--: |:--: |
| cabinet-base-server | 6 | com.fcbox.edms.terminal.api.CabinetServiceFacade#getCabinetInfo | 200|  1700 | 5000 | redis, mysql |
| cabinet-base-server | 6 | com.fcbox.edms.terminal.api.CabinetServiceFacade#getCabinetInfo4Post | 350 |  1500 | 5000 | redis, mysql |
| cabinet-state-server | 8 | com.fcbox.edms.resourcepool.api.cabinetsync.CabinetSyncFacade#asyncCellDetail | 300 |  2500 | 7500 | kafka, mongo |


#### 
 