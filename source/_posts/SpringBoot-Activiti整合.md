---
title: SpringBoot+Activiti整合
categories: 编程技术
date: 2020-01-21 06:54:04
tags: Spring Boot
keywords: [Spring Boot,Activiti]
description: Spring Boot和Activiti的整合
---

# 简介

Activiti是一个基于Apache v2开源协议的工作流引擎，工作流引擎全称是业务流程管理(Business Process Management)，一种定义了数据流转行为的流程框架，业界有很多工作流引擎，均是BPM协议的实现，例如Activiti，Flowchat等，本文基于Spring Boot，来简单实现一个图书馆的业务管理。

<!--more-->

# 环境介绍

1. Spring Boot 2.1.6.RELEASE
2. Activiti 6.0.0
3. Mysql 5.7

Activiti 6.0.0官方文档：[https://www.activiti.org/userguide/](https://www.activiti.org/userguide/)

# 代码实践

## 依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  ......
	<properties>
        <java.version>1.8</java.version>
        <activiti-version>6.0.0</activiti-version>
    </properties>
  
  <dependencies>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <!--工作流-->
        <dependency>
            <groupId>org.activiti</groupId>
            <artifactId>activiti-engine</artifactId>
            <version>${activiti-version}</version>
        </dependency>
        <dependency>
            <groupId>org.activiti</groupId>
            <artifactId>activiti-spring</artifactId>
            <version>${activiti-version}</version>
        </dependency>
        <dependency>
            <groupId>org.activiti</groupId>
            <artifactId>activiti-bpmn-layout</artifactId>
            <version>${activiti-version}</version>
        </dependency>
        <dependency>
            <groupId>org.drools</groupId>
            <artifactId>knowledge-api</artifactId>
            <version>6.5.0.Final</version>
        </dependency>
    <dependencies>
  ......
```

## 配置

对于Activiti，本质上是提供了一个通用框架来处理业务数据，即传统意义上的数据和业务分离，业务流框架自己提供了Repository来操作数据，所以这里需要自己配置单独的数据源，当然，也可以直接使用参数注入的方式将spring boot的数据源给注入进来，这里我采用单独数据源的方式。

```properties
spring.activiti.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.activiti.datasource.url=jdbc:mysql://localhost:3306/activiti?useUnicode=true&characterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull&useSSL=false&serverTimezone=GMT%2B8&nullCatalogMeansCurrent=true
spring.activiti.datasource.username=root
spring.activiti.datasource.password=root
spring.activiti.datasource.hikari.maximum-pool-size=16
spring.activiti.datasource.hikari.connection-timeout=10000
```

使用注解方式创建配置类：

```java
@Configuration
public class ActivitiConfiguration {

    @Value("${spring.activiti.datasource.driver-class-name}")
    private String driveClassName;

    @Value("${spring.activiti.datasource.url}")
    private String jdbcUrl;

    @Value("${spring.activiti.datasource.username}")
    private String userName;

    @Value("${spring.activiti.datasource.password}")
    private String password;

    @Value("${spring.activiti.datasource.hikari.maximum-pool-size}")
    private Integer maxConnections;

    @Value("${spring.activiti.datasource.hikari.connection-timeout}")
    private Integer timeout;


    /**
     * 创建流程引擎 
     *
     * @return org.activiti.engine.ProcessEngine
     * @author Joy.Liu
     */
    @Bean
    public ProcessEngine processEngine(PlatformTransactionManager transactionManager,GlobalEventListener globalEventListener) throws Exception {
        ProcessEngineFactoryBean processEngineFactoryBean = new ProcessEngineFactoryBean();
        SpringProcessEngineConfiguration processEngineConfiguration = new SpringProcessEngineConfiguration();
        processEngineConfiguration.setTransactionManager(transactionManager);
        processEngineConfiguration.setActivityFontName("宋体");
        processEngineConfiguration.setLabelFontName("宋体");
        processEngineConfiguration.setAnnotationFontName("宋体");
        processEngineConfiguration.setEventListeners(Lists.newArrayList(globalEventListener));
        processEngineConfiguration.setJdbcUrl(jdbcUrl)
                .setJdbcDriver(driveClassName)
                .setJdbcUsername(userName)
                .setJdbcPassword(password)
                .setJdbcMaxWaitTime(timeout)
                .setJdbcMaxActiveConnections(maxConnections)
                .setDatabaseSchema("activiti")
                .setDatabaseSchemaUpdate("true")
                .setAsyncExecutorActivate(true)
                .setHistoryLevel(HistoryLevel.FULL)
                .setEnableProcessDefinitionInfoCache(true)
                .setDbHistoryUsed(true);
        processEngineFactoryBean.setProcessEngineConfiguration(processEngineConfiguration);
        return processEngineFactoryBean.getObject();
    }

    /**
     * 这个bean用来执行和查询任务instance 
     *
     * @param processEngine 核心引擎
     * @return org.activiti.engine.RuntimeService
     * @author Joy.Liu
     */
    @Bean
    public RuntimeService runtimeService(ProcessEngine processEngine) {
        return processEngine.getRuntimeService();
    }

    /**
     * 这个beam用来创建和查询任务
     *
     * @param processEngine
     * @return org.activiti.engine.TaskService
     * @author Joy.Liu
     */
    @Bean
    public TaskService taskService(ProcessEngine processEngine) {
        return processEngine.getTaskService();
    }

    /**
     * 管理数据库的bean 
     * 官网说开发者一般不用……
     * @param processEngine
     * @return org.activiti.engine.ManagementService
     * @author Joy.Liu
     */
    @Bean
    public ManagementService managementService(ProcessEngine processEngine) {
        return processEngine.getManagementService();
    }

    /**
     * 查询一些静态的信息 
     *
     * @param processEngine
     * @return org.activiti.engine.RepositoryService
     * @author Joy.Liu 
     */
    @Bean
    public RepositoryService repositoryService(ProcessEngine processEngine) {
        RepositoryService repositoryService = processEngine.getRepositoryService();
        return repositoryService;
    }

    /**
     * 在流程部署过程中动态改变流程节点
     *
     * @param processEngine
     * @return org.activiti.engine.DynamicBpmnService
     * @author Joy.Liu
     */
    @Bean
    public DynamicBpmnService dynamicBpmnService(ProcessEngine processEngine) {
        return processEngine.getDynamicBpmnService();
    }

    @Bean
    public HistoryService historyService(ProcessEngine processEngine) {
        return processEngine.getHistoryService();
    }

    @Bean
    public IdentityService identityService(ProcessEngine processEngine){
        return processEngine.getIdentityService();
    }
```

主要是第一个ProcessEngine，在配置流程引擎的时候，传入了两个参数，第一个参数是Spring Boot中的事务管理，第二个参数是一个全局事件监听，关于监听后面会说到。最为常用的是`runtimeService`、`taskService`、`historyService`，其中第一个是流程运行过程中查询流程的业务类，第二个是流程运行过程中的任务类，第三个是一些历史业务类，这个历史业务，在配置为历史级别为Full的时候，会将流程过程中任务的本地变量、任务全局变量都记录下来。

基本配置就这些，配置完成以后，启动项目，框架会自动生成28个表。

* ACT_EVT_LOG：事件日志；
* ACT_GE_BYTEARRAY：流程定义和流程资源，比如bpm.xml规则文件和流程图等；
* ACT_GE_PROPERTY：系统相关属性；
* ACT_HI_ACTINST：历史流程实例，记载了流程中每个节点信息；
* ACT_HI_ATTACHMENT：历史附件；
* ACT_HI_COMMENT：历史的说明性信息；
* ACT_HI_DETAIL：历史的流程运行中的细节信息；
* ACT_HI_IDENTITYLINK：历史流程运行过程中用户关系；
* ACT_HI_PROCINST：历史流程实例，记载了流程的开始-结束等信息；
* ACT_HI_TASKINST：历史任务实例；
* ACT_HI_VARINST：历史变量实例；
* ACT_ID_GROUP：组信息；
* ACT_ID_INFO：身份信息；
* ACT_ID_MEMBERSHIP：身份信息-组信息中间表；
* ACT_ID_USER：身份信息-用户信息；
* ACT_PROCDEF_INFO：死信任务；
* ACT_RE_DEPLOYMENT：部署信息；
* ACT_RE_MODEL：模型信息；
* ACT_RE_PROCDEF：已部署的流程定义；
* ACT_RU_DEADLETTER_JOB：运行失败任务表；
* ACT_RU_EVENT_SUBSCR：运行时事件；
* ACT_RU_EXECUTION：运行时正在执行的实例；
* ACT_RU_IDENTITYLINK：运行时用户关系表；
* ACT_RU_JOB：运行时作业表；
* ACT_RU_SUSPENDED_JOB：运行时暂停任务；
* ACT_RU_TASK：运行时任务；
* ACT_RU_TIMER_JOB：运行时定时任务；
* ACT_RU_VARIABLE：运行时变量表；

这是几个表主要的描述，详细表字段的描述移步：[https://blog.csdn.net/hj7jay/article/details/51302829](https://blog.csdn.net/hj7jay/article/details/51302829)

## 需求

在使用Activiti之前，通常会自己定义一个业务表，使用业务表的id作为工作流框架中的业务key，业务key和流程实例id一一对应。

```sql
CREATE TABLE `user_business` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT '主键',
  `uid` int(11) NOT NULL COMMENT '用户id',
  `finish_time` datetime DEFAULT NULL COMMENT '结束时间',
  `reason` varchar(255) CHARACTER SET utf8mb4 DEFAULT NULL COMMENT '申请理由',
  `create_time` datetime NOT NULL COMMENT '开始时间',
  `resource_type` tinyint(2) NOT NULL COMMENT '资源类型1-纸质 2-电子',
  `process_type` tinyint(2) NOT NULL COMMENT '流程类型1-借阅(纸质) 2-归还，3借阅(电子)，4归档',
  `resource_id` int(11) NOT NULL COMMENT '资源id',
  `status` tinyint(2) NOT NULL COMMENT '状态 1-进行中 2-通过 3-拒绝 4-超时 5-撤销',
  `category_id` int(11) NOT NULL COMMENT '类目id',
  `update_time` datetime NOT NULL COMMENT '更新时间',
  `resource_str` varchar(255) DEFAULT NULL COMMENT '资源类型描述',
  PRIMARY KEY (`id`),
  KEY `idx(uid)` (`uid`) USING BTREE COMMENT 'uid的普通索引'
) ENGINE=InnoDB AUTO_INCREMENT=155 DEFAULT CHARSET=utf8;
```

### 前后端分离创建规则

如果不使用Activiti Designer来画图，也可以通过Spring MVC暴露Restful API，在Service层组装规则，例如传进来一个json格式的字符串：

```json
{
 "nodeList": [{
  "type": "start",
  "nodeName": "开始",
  "icon": "StartIcon",
  "id": "start-49f63d7d788e4afeb643e6f848114343"
 }, {
  "type": "common",
  "nodeName": "人工节点",
  "icon": "CommonIcon",
  "id": "common-af3bdb04126a4d6490368708eac18dc4"
 }, {
  "type": "end",
  "nodeName": "结束",
  "icon": "EndIcon",
  "id": "end-05fb607bccd240eab173d2ec95bea4f1"
 }],
 "linkList": [{
  "type": "link",
  "id": "link-65f8b5e455774335874ae68d81405f76",
  "sourceId": "start-49f63d7d788e4afeb643e6f848114343",
  "targetId": "common-af3bdb04126a4d6490368708eac18dc4"
 }, {
  "type": "link",
  "id": "link-5441181305fa4c268b65ea60eb1c818e",
  "sourceId": "common-af3bdb04126a4d6490368708eac18dc4",
  "targetId": "end-05fb607bccd240eab173d2ec95bea4f1"
 }],
 "status": "1",
 "processType": 4
}
```

node即为图中每一个节点，link是节点之间的联线，事先创建节点和连线的实体类，这部分不做详细介绍。然后反序列化，并创建流程图：

```java
List<ActivitiNodeElementDto> nodeElementDtoList = GSON.fromJson(jsonObject.get("nodeList"), new TypeToken<List<ActivitiNodeElementDto>>() {
        }.getType());
        List<ActivitiLineElementDto> lineElementDtoList = GSON.fromJson(jsonObject.get("linkList"), new TypeToken<List<ActivitiLineElementDto>>() {
        }.getType());
       
        BpmnModel model = new BpmnModel();
        Process process = new Process();
        model.addProcess(process);
        process.setId(1);
        process.setName("第一个流程");
        process.setDocumentation("测试");
        //添加节点
        for (ActivitiNodeElementDto node : nodeElementDtoList) {
            if (node.getType().equals(ActivitiNodeType.START_NODE.getCode())) {
                createStartEvent(node.getId(), node.getNodeName(), process);
            } else if (node.getType().equals(ActivitiNodeType.END_NODE.getCode())) {
                createEndEvent(node.getId(), node.getNodeName(), process);
            } else if (node.getType().equals(ActivitiNodeType.CONDITION_NODE.getCode())) {
                createExclusiveGateway(node.getId(), node.getNodeName(), process);
            } else {
                createUserTask(node.getId(), node.getNodeName(), node.getRoleId(), process);
            }
        }
        //添加连线
        for (ActivitiLineElementDto line : lineElementDtoList) {
            if (line.getApproval() == null) {
                createSequenceFlow(line.getId(), line.getSourceId(), line.getTargetId(), null, process);
            } else {
                createSequenceFlow(line.getId(), line.getSourceId(), line.getTargetId(), line.getApproval() ? APPROVAL_EXPRESSION : DENY_EXPRESSION, process);
            }
        }
				//绘图
        new BpmnAutoLayout(model).execute();
				//将流程定义部署到数据图
        Deployment deployment = repositoryService.createDeployment().addBpmnModel(process.getId() + ".bpmn", model).name("_deployment").deploy();

            try {
                InputStream processBpmn = repositoryService.getResourceAsStream(deployment.getId(), process.getId() + ".bpmn");
                FileUtils.copyInputStreamToFile(processBpmn, new java.io.File("processes/" + process.getId() + ".bpmn20.xml"));
            } catch (IOException e) {
                log.error("流程转储bpmn文件失败", e);
            }
      
        log.debug("部署流程：{}", deployment);
```

上面代码依赖的一些方法无非是创建各种节点：

```java
		 /**
     * 创建用户任务
     *
     * @param id      id
     * @param name    任务展示名称
     * @param roleId  审批者角色id
     * @param process 流程
     * @author Joy
     */
    private void createUserTask(String id, String name, Integer roleId, Process process) {
        UserTask userTask = new UserTask();
        userTask.setName(name);
        userTask.setId(id);
        //这个dueDate很可能没作用，但是还是设置一下吧
        userTask.setDueDate(dueDate);
        //为这个任务创建一个过期边界时间
        BoundaryEvent boundaryEvent = createBoundaryEvent(TASK_BOUNDARY_PREFIX + UUID.randomUUID().toString(), id);
        boundaryEvent.setAttachedToRef(userTask);
        userTask.setBoundaryEvents(Lists.newArrayList(boundaryEvent));
        userTask.setCandidateGroups(Lists.newArrayList(String.valueOf(roleId)));
        process.addFlowElement(userTask);
        process.addFlowElement(boundaryEvent);
    }

    /**
     * 创建连线
     *
     * @param from                连线起点
     * @param to                  连线中点
     * @param conditionExpression 表达式
     * @author Joy
     */
    private void createSequenceFlow(String id, String from, String to, String conditionExpression, Process process) {
        SequenceFlow flow = new SequenceFlow();
        flow.setSourceRef(from);
        flow.setTargetRef(to);
        flow.setId(StringUtils.isEmpty(id) ? from + "-" + to + UUID.randomUUID().toString().substring(0, 8) : id);
        flow.setConditionExpression(conditionExpression);
        process.addFlowElement(flow);
    }

    /**
     * 创建开始事件
     *
     * @param id   事件id
     * @param name 事件名字
     * @author Joy
     */
    private void createStartEvent(String id, String name, Process process) {
        StartEvent startEvent = new StartEvent();
        startEvent.setId(id);
        startEvent.setInitiator("uid");
        if (!StringUtils.isEmpty(name)) {
            startEvent.setName(name);
        }
        process.addFlowElement(startEvent);
    }

    /**
     * 创建结束事件
     *
     * @param id      id
     * @param name    事件名称
     * @param process 流程
     * @author Joy
     */
    private void createEndEvent(String id, String name, Process process) {
        EndEvent endEvent = new EndEvent();
        endEvent.setId(id);
        if (!StringUtils.isEmpty(name)) {
            endEvent.setName(name);
        }
        process.addFlowElement(endEvent);
    }

    /**
     * 创建定时边界事件
     *
     * @param id          id
     * @param attachToRef 所依附的task id
     * @author Joy.Liu (wb-lhj605829)
     */
    private BoundaryEvent createBoundaryEvent(String id, String attachToRef) {
        BoundaryEvent boundaryEvent = new BoundaryEvent();
        boundaryEvent.setId(id);
        boundaryEvent.setName("审批有效时间");
        boundaryEvent.setAttachedToRefId(attachToRef);
        //保证超时触发activiti_cancel事件
        boundaryEvent.setCancelActivity(true);
        TimerEventDefinition timerEventDefinition = new TimerEventDefinition();
        timerEventDefinition.setTimeDuration(dueDate);
        boundaryEvent.setEventDefinitions(Lists.newArrayList(timerEventDefinition));
        return boundaryEvent;
    }


    /**
     * 创建排他网关
     *
     * @param id      网关id
     * @param name    网关名称
     * @param process 流程引擎
     * @author Joy.Liu (wb-lhj605829)
     */
    private void createExclusiveGateway(String id, String name, Process process) {
        ExclusiveGateway exclusiveGateway = new ExclusiveGateway();
        exclusiveGateway.setId(id);
        exclusiveGateway.setName(name);
        process.addFlowElement(exclusiveGateway);
    }
```

执行项目，访问接口，可以在process目录下看到新创建的流程图以及对应的规则xml：

![](SpringBoot-Activiti整合/2538.png)

### 发起某个流程

```java
//设置发起人
identityService.setAuthenticatedUserId(uid);
//插入业务 业务方自己实现
userBusinessMapper.insertSelective(userBusiness);
//将获得的主键作为作为business_key
            ProcessInstance processInstance = runtimeService.startProcessInstanceByKey(processId,
                    userBusiness.getId().toString());
log.debug("开始启动流程：instance：{}，business_key:{}", processInstance, userBusiness.getId());
```

###  获取待处理任务列表

```
public List<ActivitiTodoTaskDto> getTodoTaskList(Integer uid, Integer currentPage, Integer pageSize) {
    currentPage = currentPage == null ? 1 : currentPage;
    pageSize = pageSize == null ? 10 : pageSize;
    Example example = Example.builder(UserRoleRelation.class)
            .where(WeekendSqls.<UserRoleRelation>custom().andEqualTo(UserRoleRelation::getUserId, uid))
            .build();
    List<UserRoleRelation> userRoleRelationList = userRoleRelationMapper.selectByExample(example);
    TaskQuery taskQuery = taskService.createTaskQuery();
    List<Task> taskList = taskQuery
            .taskCandidateGroup(userRoleRelationList.get(0).getRoleId().toString())
            .listPage(currentPage - 1, pageSize);
    List<ActivitiTodoTaskDto> activitiTodoTaskDtoList = Lists.newArrayList();
    for (Task task : taskList) {
        String processInstanceId = task.getProcessInstanceId();
        ProcessInstance processInstance = runtimeService.createProcessInstanceQuery()
                .processInstanceId(processInstanceId)
                .active()
                .singleResult();
        if (processInstance == null) {
            continue;
        }
        String businessKey = processInstance.getBusinessKey();
        //获取整个流程的参数
        UserBusiness userBusiness = userBusinessMapper.selectByPrimaryKey(Long.valueOf(businessKey));
        if(userBusiness != null){
            ActivitiTodoTaskDto activitiTodoTaskDto = new ActivitiTodoTaskDto();
            activitiTodoTaskDto.setId(task.getId());
            activitiTodoTaskDto.setName(task.getName());
            activitiTodoTaskDto.setUserBusiness(userBusiness);
            Map<String, Object> searchMap = new HashMap<>();
            searchMap.put("id", userBusiness.getUid());
            UserDto userDto = userMapper.userList(searchMap).get(0);
            activitiTodoTaskDto.setUserName(userDto.getNickname());
            activitiTodoTaskDto.setDepName(userDto.getDepName());
            activitiTodoTaskDto.setCategory(categoryMapper.selectByPrimaryKey(userBusiness.getCategoryId()));
            activitiTodoTaskDtoList.add(activitiTodoTaskDto);
        }
    }
    return activitiTodoTaskDtoList;
}
```

### 完成某个任务

```java
public Boolean completeTask(Boolean approval, Integer uid, String taskId, String approvalContent) {
    log.debug("ArchiveActivitiServiceImpl.completeTask params[approval:{}, uid:{}, taskId:{}]", approval, uid, taskId);
    Map<String, Object> variables = new HashMap<>(1);
    variables.put(APPROVAL, approval);
    Example example = Example.builder(UserRoleRelation.class)
            .where(WeekendSqls.<UserRoleRelation>custom().andEqualTo(UserRoleRelation::getUserId, uid))
            .build();

    List<UserRoleRelation> userRoleRelationList = userRoleRelationMapper.selectByExample(example);
    TaskQuery taskQuery = taskService.createTaskQuery();
    List<Task> taskList = taskQuery.taskCandidateGroup(userRoleRelationList.get(0).getRoleId().toString()).list();
    if (!CollectionUtils.isEmpty(taskList)) {
        Task task = taskList.stream().filter(userTask -> taskId.equalsIgnoreCase(userTask.getId())).findFirst().orElse(null);
        if (task == null) {
            log.info("该用户不存在对该任务操作权限");
            return false;
        }
        User user = userMapper.selectByPrimaryKey(uid);
        Map<String, Object> localVariable = new HashMap<>();
        localVariable.put(APPROVAL_USER_NAME, user.getNickname());
        localVariable.put(APPROVAL, approval);
        if (!StringUtils.isEmpty(approvalContent)) {
            localVariable.put(APPROVAL_CONTENT, approvalContent);
        }
        //设置本地变量，保存审批人和审批人意见
        taskService.setVariablesLocal(taskId, localVariable);
        if (approval) {
            taskService.complete(taskId, variables);
            return true;
        } else {
            Role role = roleMapper.selectByPrimaryKey(userRoleRelationList.get(0).getRoleId());
//如果审批不过，直接中断整个流程即可，从运行中表中删除，框架会自动发送一个事件            
          runtimeService.deleteProcessInstance(task.getProcessInstanceId(), role.getRoleName() + ":" + user.getNickname() + "拒绝");
            return true;
        }
    }
    return false;
}
```

上述的mapper都是业务自己的mapper，这里的处理方式除了直接使用`taskService.complete(taskId, variables);`这种方式直接完成任务以外，还可以使用`taskService.claim();`来抢占任务。

### 获取某个流程实例的节点列表

```java
//获取历史任务实例        
List<HistoricTaskInstance> historicTaskInstanceList = historyService.createHistoricTaskInstanceQuery().processInstanceBusinessKey(businessKey).finished().list();
//获取历史任务变量
List<HistoricVariableInstance> variableInstanceList = historyService.createHistoricVariableInstanceQuery().taskId(hisTask.getId()).list();
```

### 监听整个流程状态

在这个例子中，流程有以下几种类型：

```java
public enum UserBusinessStatus {

    RUNNING(1, "进行中"),
    PASS(2, "审批通过"),
    NOT_PASS(3,"审批拒绝"),
    TIMEOUT(4, "超时"),
    RECESSION(5, "撤回");
    private Integer code;

    private String description;

}
```

在一开始基础配置的时候，曾经配置一个全局监听，这里就是发挥作用的时候了。

```java
@Slf4j
@Component
public class GlobalEventListener implements ActivitiEventListener, ApplicationContextAware, InitializingBean {

    private ApplicationContext applicationContext;

    private List<AbstractEventHandler> eventHandlerList = new ArrayList<>();

    @Override
    public void onEvent(ActivitiEvent event) {
        for (AbstractEventHandler abstractEventHandler : this.eventHandlerList) {
            if (event.getType() == abstractEventHandler.getActivitiEventType()) {
                abstractEventHandler.handle(event.getProcessInstanceId());
            }
        }
    }

    @Override
    public boolean isFailOnException() {
        return false;
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        this.applicationContext = applicationContext;
    }

    @Override
    public void afterPropertiesSet() {
        String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(this.applicationContext, AbstractEventHandler.class);
        if (beanNames.length > 0) {
            for (String beanName : beanNames) {
                this.eventHandlerList.add((AbstractEventHandler) applicationContext.getBean(beanName));
            }
        }
    }
}
```

`AbstractEventHandler`：

```java
public abstract class AbstractEventHandler {

    private ActivitiEventType activitiEventType;

    public ActivitiEventType getActivitiEventType() {
        return activitiEventType;
    }

    public void setActivitiEventType(ActivitiEventType activitiEventType) {
        this.activitiEventType = activitiEventType;
    }

    protected abstract void handle(String instanceId);

}
```

作为本项目的抽象事件监听，指导手册[https://www.activiti.org/userguide/](https://www.activiti.org/userguide/)中3.17.6节中详细描述了所支持的事件类型，以及发生的时间。常用的事件有：任务审批拒绝/用户撤销、任务审批超时、流程审批通过。对于第一个事件，上面的操作是直接从运行中表删除(`runtimeService.deleteProcessInstance`)，文档中有该描述：

```
PROCESS_CANCELLED:A process has been cancelled. Dispatched before the process instance is deleted from runtime. Process instance is cancelled by API call RuntimeService.deleteProcessInstance
```

写一个监听继承抽象监听：

```java
@Slf4j
@Component
public class ProcessCancelEventHandler extends AbstractEventHandler {

    @Autowired
    private ArchiveActivitiService archiveActivitiService;

    private ActivitiEventType activitiEventType;

    public ProcessCancelEventHandler() {
        this.activitiEventType = ActivitiEventType.PROCESS_CANCELLED;
    }

    @Override
    public ActivitiEventType getActivitiEventType() {
        return this.activitiEventType;
    }

    @Override
    protected void handle(String instanceId) {
        log.info("监听到流程取消事件");
        archiveActivitiService.processCancel(instanceId);
    }
}
```

同理，超时对应`ActivitiEventType.TIMER_FIRED;`事件，这个事件会由每个任务节点绑定的过期边界`BoundaryEvent`触发，审批完成对应`ActivitiEventType.PROCESS_COMPLETED`事件，三种分别调用下面三个方法：

```java
@Override
    public void processCancel(String instanceId) {
        updateBusinessStatus(instanceId, UserBusinessStatus.NOT_PASS);
    }
@Override
    public void processTimeout(String instanceId) {
        updateBusinessStatus(instanceId, UserBusinessStatus.TIMEOUT);
    }
@Override
    public void processComplete(String instanceId) {
        VariableInstance variableInstance = runtimeService.getVariableInstance(instanceId, "approval");
        if (variableInstance != null) {
            if ((Boolean) variableInstance.getValue()) {
                log.info("获取变量：id:{},name:{},key:{},value:{}", variableInstance.getId(), variableInstance.getTypeName(), variableInstance.getName(), variableInstance.getValue());
                updateBusinessStatus(instanceId, UserBusinessStatus.PASS);
            }
        } else {
            HistoricVariableInstance historicVariableInstance = historyService.createHistoricVariableInstanceQuery().processInstanceId(instanceId)
                    .executionId(instanceId)
                    .variableName("approval")
                    .singleResult();
            if (historicVariableInstance != null && (Boolean) (historicVariableInstance.getValue())) {
                log.info("获取变量：id:{},name:{},key:{},value:{}", historicVariableInstance.getId(), historicVariableInstance.getVariableName(), historicVariableInstance.getVariableTypeName(), historicVariableInstance.getValue());
                updateBusinessStatus(instanceId, UserBusinessStatus.PASS);
            }
        }

    }
//更新业务表状态
private void updateBusinessStatus(String instanceId, UserBusinessStatus status) {
        String businessKey = "";
        ProcessInstance processInstance = runtimeService
                .createProcessInstanceQuery()
                .processInstanceId(instanceId)
                .active()
                .singleResult();

        if (processInstance == null) {
            HistoricProcessInstance historicProcessInstance = historyService
                    .createHistoricProcessInstanceQuery()
                    .processInstanceId(instanceId)
                    .finished()
                    .singleResult();
            if (historicProcessInstance != null) {
                businessKey = historicProcessInstance.getBusinessKey();
            }
        } else {
            businessKey = processInstance.getBusinessKey();
        }
        UserBusiness userBusiness = new UserBusiness();
        userBusiness.setStatus(status.getCode());
        userBusiness.setUpdateTime(new Date());

        Example example = Example.builder(UserBusiness.class).where(WeekendSqls.<UserBusiness>custom()
                .andEqualTo(UserBusiness::getId, Long.valueOf(businessKey))
                .andEqualTo(UserBusiness::getStatus, UserBusinessStatus.RUNNING.getCode()))
                .build();
        userBusinessMapper.updateByExampleSelective(userBusiness, example);
}
```

对于是用户主动撤销还是审批不通过导致的终止，可以通过变量判断，即审批不通过设置了不通过标记，而主动撤销没有该变量。

# 总结

这次的工作流整合中遇到不少的问题，但是要始终牢记工作流只是简化了对数据的操作而已，最终业务方依然是自己的代码，7.x系列文档还不齐全，感谢大神咖啡兔的博客：

[http://www.kafeitu.me/activiti-in-action.html](http://www.kafeitu.me/activiti-in-action.html)以及好友**Hacken丶**分享的部分公司内部处理方案。