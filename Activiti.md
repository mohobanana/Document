# 1.spring-activiti环境配置

## maven

```xml
<dependency>
    <groupId>org.activiti</groupId>
    <artifactId>activiti-engine</artifactId>
    <version>${activiti.version}</version>
</dependency>
<dependency>
    <groupId>org.activiti</groupId>
    <artifactId>activiti-spring</artifactId>
    <version>${activiti.version}</version>
</dependency>
<dependency>
    <groupId>org.activiti</groupId>
    <artifactId>activiti-rest</artifactId>
    <version>${activiti.version}</version>
</dependency>
```

## acidity.cfg.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="processEngineConfiguration" class="org.activiti.spring.SpringProcessEngineConfiguration">
        <!-- 配置第三方连接池 -->
        <property name="dataSource" ref="dataSource"/>
        <!--
            databaseSchemaUpdate: 设置流程引擎启动和关闭时如何处理数据库表
            false:(默认) 检查数据库的版本和依赖库的版本，如果版本不匹配就抛出异常。
            true：构建流程引擎时，执行检查，如果需要就执行更新，如果不存在就创建
            create-drop：构建流程引擎时创建数据库表，关闭流程引擎时就删除这些表
         -->
        <property name="databaseSchemaUpdate" value="true"/>
        <!-- 是否启动任务调用 -->
        <property name="jobExecutorActivate" value="false"/>
        <property name="transactionManager" ref="transactionManager" />
        <property name="deploymentResources">
            <list>
                <value>classpath*:META-INF/banana/diagram/*.bpmn</value>
            </list>
        </property>
        <property name="activityFontName" value="宋体"/>
        <property name="labelFontName" value="宋体"/>
    </bean>
</beans>
```

***

# 2.API

## Service

### 	service获取

​		1.在spring上下文中进行如下配置，然后使用@Autowired注入

```xml
<bean id="processEngine" class="org.activiti.spring.ProcessEngineFactoryBean">
    <property name="processEngineConfiguration" ref="processEngineConfiguration" />
</bean>
<bean id="repositoryService" factory-bean="processEngine"
      factory-method="getRepositoryService" />
<bean id="runtimeService" factory-bean="processEngine"
      factory-method="getRuntimeService" />
<bean id="taskService" factory-bean="processEngine"
      factory-method="getTaskService" />
<bean id="historyService" factory-bean="processEngine"
      factory-method="getHistoryService" />
<bean id="managementService" factory-bean="processEngine"
      factory-method="getManagementService" />
<bean id="IdentityService" factory-bean="processEngine"
      factory-method="getIdentityService" />
```
​		2.使用processEngine内部方法获取

```java
ProcessEngine processEngine = ProcessEngines.getDefaultProcessEngine();

RuntimeService runtimeService = processEngine.getRuntimeService();
RepositoryService repositoryService = processEngine.getRepositoryService();
TaskService taskService = processEngine.getTaskService();
ManagementService managementService = processEngine.getManagementService();
IdentityService identityService = processEngine.getIdentityService();
HistoryService historyService = processEngine.getHistoryService();
FormService formService = processEngine.getFormService();
DynamicBpmnService dynamicBpmnService = processEngine.getDynamicBpmnService();
```

___

### 	service介绍

#### 	RepositoryService

相关表：

​		ACT_GE_BYTEARRAY（流程定义部署的相关信息）
​		ACT_RE_DEPLOYMENT（存放流程定义显示名和部署时间）
​		ACT_RE_PROCDEF（部署流程定义的属性信息，当key相同时，新部署的流程定义为升级）

作用：

- 管理流程定义文件xml及静态资源的服务
- 对特定流程的暂停和激活
- 流程定义启动权限管理



##### 						-**流程定义部署**

​		使用bpmn的id作为key，在表act_re_procdef中新增一条记录，重复key以version作为区分

```sql
select * from act_re_procdef;
```

​		**1.classPath方式部署**：classPath一般用于开发阶段，在真正的产品使用中，很少会用到这种方式。

```java
//定义资源文件的classPath
String bpmnClassPath = "META-INF/banana/diagram/testDiagram.bpmn";
//创建部署构建器
DeploymentBuilder deploymentBuilder = repositoryService.createDeployment();
//添加资源
deploymentBuilder.addClasspathResource(bpmnClassPath);
//执行部署
deploymentBuilder.deploy();
```

​		**2.InputStream方式部署**：使用inputStream部署时，需要额外的参数是资源的名称。以上的代码是从一个绝对路径去读取文件，生成inputStream输入流。这种方式可能是在项目中最广泛使用的，比如通过上传bpmn文件来部署，或者给出文件URL地址来部署。

```java
InputStream in = this.getClass().getClassLoader().getResourceAsStream("META-INF/banana/diagram/bananaDiagram.bpmn");
//读取filePath文件作为一个输入流
repositoryService.createDeployment().addInputStream("bananaDiagram.bpmn",in).deploy();
```

​		**3.zip/bar格式的压缩包方式部署**：压缩包中有多少个资源文件，则就可以部署多少个，其实类似于classPath的批量。

```java
InputStream zipStream = getClass().getClassLoader().getResourceAsStream("META-INF/banana/diagram/bpmn.zip");
        repositoryService.createDeployment().addZipInputStream(new ZipInputStream(zipStream)).deploy();
```



##### 				-**查询**

​		通过ProcessDefinitionQuery类对已部署流程进行查询

```java
ProcessDefinitionQuery processDefinitionQuery = repositoryService.createProcessDefinitionQuery();
    List<ProcessDefinition> processDefinitionList = processDefinitionQuery.listPage(0,10);
```



##### 				-**流程定义删除**

​		deleteDeployment方法有两个参数，第一个是部署ID，第二个参数true表示删除时，会同时把流程相关的流程数据(包括运行中的和已经结束的流程实例)一并删除掉。

```java
repositoryService.deleteDeployment(String processDefinitionId, boolean flag);
```



##### 			-**流程定义挂起与激活**

​		**1.流程定义的激活**

```java
repositoryService.activateProcessDefinitionById(String processDefinitionId, boolean flag, null);
```

​		**2.流程定义的挂起**：流程定义的挂起，是否影响已经启动的流程实例继续可通过第二个参数配置

```java
repositoryService.suspendProcessDefinitionById(String processDefinitionId, boolean flag, null);
```

___

#### 	RuntimeService

相关表：

​		ACT_RU_EXECUTION（流程实例表）
​		ACT_HI_PROCINST（历史表）

作用：

- 启动流程及对流程数据的控制
- 流程实例(ProcessInstance)与执行流(Execution)的查询
- 触发流程操作,接收消息和信号



##### 				-**流程实例与执行流的概念**
​		在Activiti中，启动了一个流程后，就会创建一个流程实例(ProcessInstance),简单来说流程实例就是根据一次（一条）业务数据用流程驱动的入口
​		Execution的含义就是一个流程实例（ProcessInstance）具体要执行的过程对象。
​		两者的对象映射关系：ProcessInstance（1）—>Execution(N)，其中N >= 1。
​		每个流程实例至少会有一个执行流(execution)，如果流程中没有分支，则N=1，如果流程中出现了分支，则N>1



##### 				-**流程实例启动**

​		RuntimeService提供了很多启动流程的API，并且全部的命名规则为startProcessInstanceByXX，比如按照流程定义key值启动的，按照流程定义Id启动的等等。

```java
// 根据流程定义id启动流程实例
ProcessInstance processInstance = runtimeService.startProcessInstanceById(String processDefinitionId);
// 根据流程定义key启动流程实例
ProcessInstance processInstance = runtimeService.startProcessInstanceByKey(String processDefinitionKey);
// 启动流程实例时带上参数，参数在该实例全局变量中
Map<String,Object> variables = new HashMap<String, Object>();
variables.put("key1","value1");
ProcessInstance processInstance = runtimeService.startProcessInstanceByKey(String processDefinitionKey,Map<String,Object> variables);
```

​		同时在**formService**中也提供了流程启动的方法：

```java
ProcessInstance submitStartFormData(String processDefinitionId, Map<String, String> properties);
ProcessInstance submitStartFormData(String processDefinitionId, String businessKey, Map<String, String> properties);
```



##### 				-**查询**

​		**1.查询执行流**

```java
List<Execution> executionList = runtimeService.createExecutionQuery().processDefinitionKey(String processDefinitionKey).list();
```

​		**2.查询实例**

```java
//根据流程定义Key值查询正在运行的流程实例
List<ProcessInstance> processInstanceList = runtimeService.createProcessInstanceQuery().processDefinitionKey(String processDefinitionKey).list();
//查询激活的流程实例
List<ProcessInstance> activateList = runtimeService.createProcessInstanceQuery().processDefinitionKey(String processDefinitionKey).active().list();
//查询挂起的流程则是
List<ProcessInstance> suspendList = runtimeService.createProcessInstanceQuery().processDefinitionKey(String processDefinitionKey).suspended().list();
//根据变量来查询
List<ProcessInstance> varList = runtimeService.createProcessInstanceQuery().variableValueEquals(String key,String value).list();
```



##### 			-**流程实例删除**

​		影响表：ACT_HI_PROCINST，ACT_RU_TASK

```java
runtimeService.deleteProcessInstance(String processInstanceId,String deleteMessage);
```



##### 			-**流程实例挂起与激活**

​		**1.流程实例的挂起**

```java
runtimeService.suspendProcessInstanceById(pString processInstanceId);
```

​		**2.流程实例的激活**

```java
runtimeService.activateProcessInstanceById(String processInstanceId);
```

___

#### 	TaskService

相关表：

​		ACT_RU_TASK（执行任务表）
​		ACT_HI_TASKINST（历史任务表）

任务分类：

​		userTask（用户任务）
​		servicetask（服务任务）



##### 		-**管理Task与流程控制**

- Task对象的创建和删除
- 查询Task,驱动Task节点完成执行
- Task相关参数变量variable设置

​		使用taskService设置局部变量后，局部变量的作用域只限于当前任务内，如果任务结束后，那么局部变量也就随着消失了。

```java
// 查询task
Task task1 = taskService.createTaskQuery().singleResult();
 // 设置全局变量
taskService.setVariable(String taskId,String key,String value);
// 设置局部变量
taskService.setVariableLocal(String taskId,String key,String value);

// 获取全局变量
Map<String,Object> a = taskService.getVariables(String taskId);
// 获取局部变量
Map<String,Object> b = taskService.getVariablesLocal(String taskId);
// 获取全局变量
Map<String,Object> c = runtimeService.getVariables(String processInstanceId);
```

​		除了直接设置变量外，在任务提交的时候也可以附带变量，即使用`taskService.complete()`,这个complete方法多个重载方法：

```java
public void complete(String taskId);
public void complete(String taskId, Map<String, Object> variables);
public void complete(String taskId, Map<String, Object> variables, boolean localScope) ;
//6.0后新增瞬时变量transientVariables,瞬时变量仅能在下一个等待状态之前被获取,不会被持久化
public void complete(String taskId, Map<String, Object> variables, Map<String, Object> transientVariables);
```

​		其中taskId(对应act_ru_task中的id_)，variables（下一次任务所需要的参数）,作用是完成这一次任务，并且下一步任务需要流程变量的。要注意的是localScope这个参数：localScope（存储范围：本任务） 。当这个布尔值为true表示作用范围为当前任务，当任务结束后，再也取不到这个值了，act_ru_variables这个表中也没有这个参数的信息了；如果为false表示这个变量是全局的，任务结束后在act_ru_variables表中仍然能查到变量信息。



##### 	-**设置Task权限信息**

- 候选用户candidateUser和候选组candidateGroup

- 指定拥有人Owner和办理人Assignee

- 通过claim设置办理人（签收）


​	**Assignee（受理人）、Owner（委托人）与Candidate（候选）**

**Assignee（受理人）**：task任务的受理人，就是执行TASK的人，这个又分两种情况（有值，NULL）
  1）有值的情况：XML流程里面定义的受理人，TASK会直接填入这个人；
  2）NULL：XML没有指定受理人或者只指定了候选，此时可以使用签收功能去指定受理人；
**Owner（委托人）**：受理人委托其他人操作该TASK的时候，受理人就成了委托人Owner，其他人就成了受理人Assignee

​	**设置候选用户/候选组**

```java
taskService.addCandidateUser(String taskId, String userId)
```

​	**签收与取消**

一个任务只能被签收一次

```java
taskService.claim(String taskId,String assignee);
//取消
taskService.setAssignee(String taskId,null);
```



##### **-设置Task附加信息**

- 任务附件（Attachment）创建与查询
- 任务评论（Comment）创建与查询
- 事件记录（Event）创建与查询



​	**任务附件**

```java
/**
 * 附件创建
 */
//创建任务附件，attachmentType为附件类型，由调用者定义；taskld为任务ID；processlnstanceld为流程实例ID；attachmentName为附件名称；attachmentDescription为附件描述；url为该附件的 URL地址。
createAttachment(String attachmentType, String taskId, String processlnstanceId, String attachmentName, String attachmentDescription, String url);
//输入流创建任务附件
createAttachment(String attachmentType, String taskId, String processlnstanceId, String attachmentName,String attachmentDescription, InputStream content);
/**
 * 附件查询
 */
//根据流程实例ID查询该流程实例下全部的附件，返回Attachment集合。
getProcesslnstanceAttachments(String processlnstanceld);
//根据任务ID查询该任务下全部的附件，返回Attachment集合
getTaskAttachments(String taskId);
//根据附件的ID查询附件数据，返回一个Attachment对象。
getAttachment(String attachmentId);
//根据附件的ID获取该附件的内容，返回附件内容的输入流对象，如果调用的是第二个createAttachment方法（传入附件的InputStream），那么调用该方法才会返回非空的输入流，否则将返回 null。
getAttachmentContent(String attachmentId);
/**
 * 附件删除
 */
//只会删除附件表中的数据及相应的历史数据，不会删除资源表中的数据。
deleteAttachment(String attachmentId)
```

​	**任务评论与事件记录**

```java
/**
 * 评论新增
 */
addComment(String comment);
/**
 * 评论查询
 */
//根据评论数据ID查询评论数据。
getComment(String commentId);
//根据任务ID查询相应的评论数据。
getTaskComments(String taskId);
//根据任务ID查询相应的事件记录。
getTaskEvents(String taskId);
//根据流程实例ID查询相应的评论（事件）数据。
getProcesslnstanceComments(String processlnstanceId);
```

___









