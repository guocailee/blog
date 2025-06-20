---
{"dg-publish":true,"permalink":"/Program/Framework/Qunar 酒店基础数据重构DDD落地实践/","noteIcon":"","created":"2025-03-06T21:28:25.970+08:00"}
---


**李全党**

2021年6月入职去哪儿网，酒店供应链高级技术经理、业务架构SIG成员、公司级内训师，目前负责酒店基础信息业务。主导搭建并落地多个DDD项目，并对高并发、分布式服务高可用，有建设优化经验，2021年落地公司“大主站+微服务”战略，并获得公司“金项奖”技术类三等奖，曾在QCon做过技术分享。

随着集团战略方向调整与业务重组，酒店供应链也面临全新的调整。酒店基础数据业务系统是从国际团队接手，属于10年前系统架构，涉及20多个微服务，架构老旧、系统耦合严重及业务边界模糊，加上业务的快速发展，导致系统灵活性不足及无法快速承接产品需求，产研合作出现效率问题。因此，酒店供应链技术侧结合酒店BU及公司相关成功案例，在2022年初主动发起基于DDD思想的技术架构调整，完整落地战略、战术设计及系统实现，本次重点介绍重构落地过程、设计原则及总结，另外阅读本文需要对DDD的基本概念及流程有一定基础，基本概念可以参考往期技术沙龙（链接见文末）。

**二、问题分析**

**（一）业务需求复杂**
-------------

酒店基础信息业务系统最初是为去哪儿独立业务设计，主要包括酒店聚合、房型聚合、图片及城市等相关信息（见下图），最初业务划分比较清晰。2015年携程战略收购去哪儿网后，业务战略也随之发生变化，但当时并未对系统进行隔离和重构，随着需求迭代的不断演化，业务逻辑变得越来越复杂，加上产品经理的流动性较大，酒店基础数据PM、DEV、QA团队缺少业务专家，大家对于原始需求缺少理解，部分业务不确定能否下线等等。

![](https://mmbiz.qpic.cn/mmbiz_png/YE1dmj1Pw7mzWKVAicdPXgb1LiaGOpovRPV4g7JgovjlLfgEria7Ze77UXbQplsHeeBT3PTkWnlEXA7Z4Ifv3VpLg/640?wx_fmt=png)

**（二）系统过度耦合**

酒店基础数据涉及微服务20多个，系统之间耦合严重，模块彼此关联，我们的系统越来越冗杂。有时修改一个很小的产品需求，光回溯该需求涉及需要修改的系统及功能点就需要达到“天”级别，更别提修改带来的不可预知的影响面，无论还是产品、技术需求还是工单问题排查等，都给组内同学带来较高学习成本和开发成本。

下图是我们酒店基础信息日常开发中的一个常见的系统耦合案例。

![](https://mmbiz.qpic.cn/mmbiz_png/YE1dmj1Pw7mzWKVAicdPXgb1LiaGOpovRP7lAiaM8rUZDDA12icDSbs1IO572wiaPo54nYYkTIEQfWiawCadyQNlBh1w/640?wx_fmt=png)

这个案列是要完成的业务功能是支持运营人员手工添加酒店图片，当时预估工时时长8pd，实际工时11pd。上图看出从运营开始上传到我们最终将图片外网展示，整体流程涉及到了4个系统。

以下这段话是当时开发这个产品需求的同学的“真情告白”：

1、需求及系统设计阶段。接到TL的排期并阅读产品PRD，发现涉及4个业务系统，深入系统内部发现业务逻辑复杂及系统耦合严重，熟悉系统及编写设计方案花费3pd；

2、系统开发阶段。按照设计进行功能开发发现，除正常多业务系统调用外，业务系统内部存在兜底定时任务，需要进一步熟悉其作用、数据存储及影响范围，花费6pd；

3、系统自测阶段。系统自测涉及多个系统间的耦合及不同的数据库版本，自测花费2pd。

如果我们的系统经过精心设计，这个案例是不需要8pd，更别说最终上线花费11pd。我们可以看出系统过度耦合，不但降低我们的开发效率，而且对于开发人员不够友好，这种情况对于人员的稳定性也造成很不好的影响，所以我们必须要做出改变。

**三、为什么选择DDD**

DDD有很多优势，我们站在EA角度，可以绑定业务架构和系统架构，作为中间层，将问题域与应用架构相剥离；我们站在软件复杂度角度，可以有效解决业务复杂度和软件复杂度问题；还比如DDD可以有效的从业务视角对软件系统进行拆解，是微服务划分最好的实践等等，作为基础数据重构，结合我们上面提到的业务需求复杂和业务过度耦合问题，我们主要从解决软件复杂度和微服务拆分角度考虑，最终选择DDD作为我们项目落地的指导原则。

**（一）软件复杂****度应对**

Eric Evans 认为“应用程序最主要的复杂性并不在技术上，而是来自领域本身、用户的活动或业务”。因而，领域驱动设计关注的焦点在于领域和领域逻辑，因为软件系统的本质其实是给客户（用户）提供具有业务价值的领域功能。那么DDD是如何应对软件复杂度的呢？我们可以粗略的归为三类，分别是分而治之、关注点分离和统一语言。

1、分而治之。比起单体架构将所有功能都糅合在一起，DDD通过在其战略设计层面对限界上下文、上下文地图的划分来做到这一点。各个业务领域内关注自身业务能力的内聚，明确分工，不被其他领域业务侵蚀，这不仅符合“高内聚、低耦合”的架构思路，更是与微服务拆分的思想不谋而合；

2、关注点分离。DDD使得领域模型与存储模型分离，业务复杂度与技术复杂度分离；

3、统一语言。我们大部分需求是横跨多个团队，需求传递低效，需要反复沟通，方案产出效率低，而统一语言使得产研在业务概念、理解等方面达成一致，降低沟通和理解成本。

![](https://mmbiz.qpic.cn/mmbiz_png/YE1dmj1Pw7mzWKVAicdPXgb1LiaGOpovRPyNKqGxicXmxEqheYUFVuPg4nJoibCgd6PV9ib17bbmMgGpibNx2tsz4Mwg/640?wx_fmt=png)

**（二）微服务架构模式的最佳实践**

微服务有9大特征，我们这里不赘述。由于DDD可以有效的从业务视角对软件系统进行拆解，并且DDD特别契合微服务的一个特征：围绕业务能力构建。所以用DDD拆分出来的微服务是比较合理的而且能够实现高内聚低耦合。我们之前会因为微服务不知如何拆分讨论上好几天，其实根本原因是不知道边界在什么地方，而使用DDD对业务分析的时候：

1、使用聚合把关联性强的业务概念划分在同一个限界上下文，并限定聚合和聚合之间只能通过聚合根来访问

2、聚合基础之上根据业务相关性，业务变化频率，组织结构等等约束条件来定义限界上下文。

DDD的聚合和限界上下文，使得拆分微服务不再困难。

**四、演化式技术改造**

在我们确定使用DDD作为指导进行项目落地，一个难题是我们如何推进多业务系统重构上线，如果采用完全重做的方式，摒弃原有的系统重新开发，虽然可以快速有效摆脱原有系统的历史技术债，但是也会丢失那些细小而繁杂的业务逻辑，为整个系统的改造带来巨大的项目风险，特别是像酒店聚合、房型聚合等涉及多种算法的业务，严重时会导致高额赔付。

最终我们采用的是**演化式技术改造**，即利用重构在原有的系统上逐步改造。即在原有系统的基础上，通过一步一步演进式的代码调整，逐步达到技术改造的目标。每一步重构，对于核心业务功能保持外部功能不变，重点调整程序内部结构，通过内部程序结构的优化，业务代码逐渐与各个层次、各种技术解耦。可以看出，演化式技术改造的目标是实现业务代码与技术框架的解耦，通过重构将长周期的改造过程，通过业务域、重要程度等划分成一个短周期的重构，保障每次重构的正确性。

酒店基础数据重构包含酒店业务、房型业务及图片业务，我们采用分阶段上线，优先上线酒店业务。那么对于酒店业务，其又包含了酒店聚合、酒店静态信息、酒店抓取等多个子业务，为保障重构业务输出正确性及阶段成果，我们将酒店业务又划分两阶段上线和功能验证，正是这样的过程，保障了在改造过程中，虽然修改了既有代码，但没有影响既有功能，使得改造更加平稳的进行下去。经过这一系列改造，系统的业务代码与技术框架解耦了，接口层建立起来，就可以从容地开展真正的技术改造。

![](https://mmbiz.qpic.cn/mmbiz_png/YE1dmj1Pw7mzWKVAicdPXgb1LiaGOpovRPGzeg5aoSh3via4DbRnN8CxVcYGycHMZQqzTE5f7wicKQIrBDpUyQia4ZQ/640?wx_fmt=png)

**五、DDD实践及设计原则**

**（一）完整的DDD落地流程框架**
-------------------

![](https://mmbiz.qpic.cn/mmbiz_png/YE1dmj1Pw7mzWKVAicdPXgb1LiaGOpovRPtSOMIw0qwb5Pr81E3VKSMXvMh2picjcZiaZdjd78BosMbSguTasKbIsw/640?wx_fmt=png)

上图为整体DDD落地流程框架，也是thought works强力推荐的流程，那么我们本篇文章并不打算去讲解流程的每个细节，而是重点介绍我们在酒店基础信息重构过程中，所积累的落地时间经验和原则，希望通过这些经过抽象和总结的经验，能够给各位带来一些收获和思考，具体内容请接着往下看。

**（二）DDD落地实践原则**

*   ### **产研沟通，定位愿景**
    

定位愿景的主要目的是对产品的顶层价值设计，对产品目标用户、核心价值、差异化竞争点、痛点等策略层信息在团队层面达成共识，这也是我们做这个事情的根本。而对于部分的DDD分享文章或者我们真正在做项目落地时，缺少关注我们的项目定位、愿景是什么，项目前期没有关注，会导致我们在划分子域时，不清楚如何确定是核心域、支撑域还是通用域等，以及资源安排没有侧重点，愿景的定位非常重要，尤其在我们新项目落地。

我们常用的定位愿景所使用的方法叫“电梯演讲”， 领域专家与项目团队一起思考，我们做的项目业务范围、目标用户、核心价值和愿景，我们与同类产品的差异和优势在哪里？整个过程是统一项目建设方向和团队思想的过程，当然如果我们的项目是一个纯粹的后端业务系统，并不是所有点都涉及，也可以跳过，比如下图所示的“关键的优点，难以抗拒的使用理由”，对于酒店静态信息而言，我们的使用用户更多是我们公司内部，那么就没有难以抗拒的使用理由了。

![](https://mmbiz.qpic.cn/mmbiz_png/YE1dmj1Pw7mzWKVAicdPXgb1LiaGOpovRPc8LwZHtEcTyicTXuvC2EDIXkR4rpFXnYFpvkoNic0ecyIbCs4MqlElIA/640?wx_fmt=png)

*   ### **产研融合，提炼问题域**
    

理解一个复杂问题域以便创造简单且有用的模型，需要深入详尽的知识以及深刻的见解，这些需要产、运、研、测共同协作得到。只有通过协作及共享对问题域的理解，才能有效设计领域模型以应对业务的挑战，这样也能具备足够的灵活性应对新出现的需求。初期我们与产运一起，通过线上画板工具（初期推荐使用BeeArt，也可使用ProcessOn）及公司白板，开展事件及命令风暴，一起探讨应用程序的应用场景。这一过程是所有参与者进行花火碰撞，获得领域的深刻见解的催化剂，通过风暴我们提炼出领域知识，重新梳理业务流程，并形成通用语言（可以包括显示的业务规则、领域名词解释等，形成统一的思维地图），达成共识。

下图右半部分为“酒店静态信息业务流程”，我们本次重构重新对原有“蜘蛛网”式的业务流程进行梳理，明确业务流阶段、价值及相应显式规则，与产运沟通确认20次（微信、邮件、线下会议、腾讯会议等）。通过产研融合，提炼问题域后，确认线上低价值或无用业务占比线上总业务用例的41.9%（原始业务用例：222个，重构后保留业务用例：129个，共下掉93个业务）。

![](https://mmbiz.qpic.cn/mmbiz_png/YE1dmj1Pw7mzWKVAicdPXgb1LiaGOpovRP5icBspKsdSia8YZhgQhcb11g7nMqkUADibMntrorIZiasVmibJySmW48Zkw/640?wx_fmt=png)

*   **集中精力，专注核心域**
    

酒店基础数据涵盖酒店、房型、图片及位置区域等多个问题域，我们需要将问题域分而治之来降低复杂性，因为较小的模型可以在子域的上下文中更容易被理解。并通过划分核心域、通用域及支撑域来决定研发策略（比如：视频、图片及城市等通用域业务简单且需求相对稳定，并未进行重构，保持现状）及资源配备。相信大家在进行项目开发的时候，一定会觉得时间紧、任务重及资源不足等等，那么这个时候我们更要关注要做的项目，他的核心价值是什么（可以参考领域愿景），因为他代表团队或组织的价值所在，所以我们要专注核心领域，安排核心开发人员及做好跟踪，确保核心领域按照预期推进。

![](https://mmbiz.qpic.cn/mmbiz_png/YE1dmj1Pw7mzWKVAicdPXgb1LiaGOpovRPvibrMuElysDYPj1QmdY2lAhkon6ia1Pyfic3JqyoKUJC5Cx7E5iaAASJaA/640?wx_fmt=png)

*   ### **使用限界上下文保护领域模型完整**
    

限界上下文拥有从展现层到领域逻辑层，再到持久化，甚至到数据存储功能的垂直切片，那么产品的概念可以存在每个限界上下文中，并且包含仅对该上下文普遍存在的特性和逻辑。任何限界上下文中的变化不再具有对其他有界上下文的影响，因为限界上下文或子域是隔离的。限界上下文具备概念上的独立性，一个限界上下文内的子概念的解释和目的都不应该超出上下文的边界，若出现以下依赖关系，需要思考是否存在未澄清的问题：

a. 双向依赖:上下文之间缺少一层未被澄清的上下文，或者两个上下文其实可被合为一个；

b. 循环依赖:任何一个上下文发生变更，依赖链条上的上下文均需要改变；

c. 过长的依赖:自身依赖的信息不能直接从依赖者获取到，需要通过依赖者从其依赖的上下文获取并传递，依赖链 路过长，依赖链条上的任何一个上下文发生变更，其链条后的任何一个上下文均可能需要改变。

![](https://mmbiz.qpic.cn/mmbiz_png/YE1dmj1Pw7mzWKVAicdPXgb1LiaGOpovRPHT6M7DvZg5iaroBhdav5sX9Yy1J5tTEMZPgaRoalGaT3dlArH71XYJw/640?wx_fmt=png)

另外我们要把握好领域职责，领域之外的事不要管，同时领域之间的数据交互，需要通过“防腐层”将领域外的对象转换为领域内对象。举个例子：下图是我们酒店的抓取解析流程，重构前不仅做了抓取和解析，还做了特殊属性处理、视频处理等非自己领域的事情，所以我们重构后，将这两个事情让给了“酒店静态信息上下文”去处理，并形成抓取解析上下文，只做一件事情：从外部抓取数据，并将数据转换为Qunar内部对象。

![](https://mmbiz.qpic.cn/mmbiz_png/YE1dmj1Pw7mzWKVAicdPXgb1LiaGOpovRPR4FX5r6qRU6lMicZ6BWzymeibtibspG4hbfaV03L9aREH6d67vspZ7klg/640?wx_fmt=png)

*   ### **应用程序架构落地COLA**
    

目前比较流行的应用架构，都会遵循一些共同模式，不管是六边形架构、洋葱圈架构、整洁架构、还是COLA架构，都提倡以业务为核心，解耦外部依赖，分离业务复杂度和技术复杂度。我们最终选择了COLA架构，有几点原因：

*   #### **_明确的分层架构_**
    

所有的复杂系统都会呈现出层级结构，应用系统处理复杂业务逻辑也应该是分层的，下层对上层屏蔽处理细节，每一层各司其职，分离关注点

1、适配层：针对不同端、协议的适配，包括酒店基础信息命令执行、查询及适配返回等；

2、应用服务层：酒店基础信息业务用例识别，并负责调用领域层能力，对用例进行组装、编排及返回结果；

3、领域服务层：提供酒店基础信息相关业务能力，并针对复杂或跨聚合业务提供领域服务能力，供应用服务层调用；

4、基础设施层：主要处理技术细节问题的处理，包括领域外部服务访问防腐层实现，数据库DB、缓存等持久化。

![](https://mmbiz.qpic.cn/mmbiz_png/YE1dmj1Pw7mzWKVAicdPXgb1LiaGOpovRPGCHoaKuQmEWic3O6H21E1fSvfq8f9gnSEtIiaKlPXDgpLhQ08re4S7Sg/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/YE1dmj1Pw7mzWKVAicdPXgb1LiaGOpovRPzSKoKGl691JOr9F1aZmB6AxYAJqYAgMxUg13btotiaBcYYKf9Mzu2HQ/640?wx_fmt=png)

扩展：我们在整个战术设计的过程中，利用奥卡姆剃刀原理（是指如无必要，勿增实体，即“简单有效原理”）的思想，引入DP（Domain Primitive 是 Value Object 的进阶版，在原始 VO 的基础上要求每个DP拥有概念的整体，而不仅仅是值对象）并把实体属性归类，为 VO 的 Immutable 基础上增加了 Validity 和无状态行为，防止将过多的属性拍平到实体。

![](https://mmbiz.qpic.cn/mmbiz_png/YE1dmj1Pw7mzWKVAicdPXgb1LiaGOpovRPUn3APMlQrb2dY4Mn6xLgj3r4ALET4sibf663FyeliaL91Yq2ndM1Y1icA/640?wx_fmt=png)

*   #### _**符合演进式架构设计**_
    

分层是属于大粒度的职责划分，我们有必要往下再down一层，细化到包结构的粒度，才能更好的指导我们的工作。COLA架构除了有分层规范以外，对每一层内部的包结构也有明确规范，即“聚合分包，功能分类”，这样的设计可以更好的应对未来需求变化的不确定性，符合演进式架构（演进式架构就是以支持增量的、非破坏的变更作为第一原则，同时支持在应用程序结构层面的多维度变化。那如何判断微服务设计是否合理呢？随着业务的发展或需求的变更，在不断重新拆分或者组合成新的微服务的过程中，不会大幅增加软件开发和维护的成本，并且这个架构演进的过程是非常轻松、简单的）。

![](https://mmbiz.qpic.cn/mmbiz_png/YE1dmj1Pw7mzWKVAicdPXgb1LiaGOpovRPfFXZH9JiaPQo4FRW3VZTKiaYeuF9dab732GibBLvPt4ht90HqialnEGTibw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/YE1dmj1Pw7mzWKVAicdPXgb1LiaGOpovRPEz25uPNl7p4aRqvJ0Bt0hhcGo7SpbFDibWfmgQa9xQGaD9gib6my53Mg/640?wx_fmt=png)

COLA架构，本质上没有什么严格的约束，对于业务代码，还是有非常好的指导建议，不一定非要严格的按照框架的要求来执行，但一定要有规范的思想，这才是核心的。实际开发中，我们并没有完全按照CLOA架构的指导要求来，比如扩展组件并没有使用，适合自己的才最重要。相信只要我们做好分模块，分层次，做好命名规范+一定的充血模型，代码就能做到简洁易懂。

**六、案例成果总结**

**（一）案例重构成果**
-------------

1、业务复杂度方面，我们共梳理222个业务用例，重构后保留129个，共下掉93个，业务平均下掉41.9%，对于降低产运研业务学习带来较大帮助；

2、瘦身服务及减链路方面，酒店基础信息业务涉及21个应用微服务，通过DDD领域划分后下降到13个，微服务减少33%，对应代码下降情况统计，平均代码下掉58.3%，大大降低研发学习及硬件成本；

3、业务专家方面，通过事件、命令风暴，对于项目成员对酒店基础信息业务更加有全局观，与产研形成统一语言和知识，减少产、运、测、研沟通成本，在业务专家方面增加2名研发业务专家，让我们更多站在业务视角去思考和解决问题，而不是来了需求首先想到的是工时问题；

4、效率提升方面，问题处理，工单处理下降50%。

![](https://mmbiz.qpic.cn/mmbiz_png/YE1dmj1Pw7mzWKVAicdPXgb1LiaGOpovRPzEgcDBHnS7WpjIbe9PK3JoknYDCv5XNSkH6jQQCrea1w7Jpx9O2dCA/640?wx_fmt=png)

**（二）项目重构常见问题**
---------------

*   ### **没有领域专家怎么办？**
    

我们都知道，业务领域专家和开发团队之间的协作是DDD必不可少的部分，特别是战略、战术设计阶段。不过，寻找到当前业务领域中是专家以及能够为问题域提供深刻见解的人，可谓是少之又少。那么如果没有领域专家怎么办呢？作为替代，可以寻找对于我们当前工作领域具备多年经验和理解的产品所有者、用户、研发或其他任何人，不必在意他们是不是权威，是不是管理者等等。

以我们当前的重构项目为例，酒店基础信息产品流动率高，没有对应的产品业务专家，最终我们寻找到具有多年工作经验的资深研发和业务QA作为我们的领域专家，最终取得的效果也是非常不错的。我们推荐的业务领域专家顺序为：业务产品团队->技术负责人->业务开发负责人->业务QA。

*   ### **简单问题复杂化**
    

1、DDD的价值在于帮助管理显著优势的复杂问题域，请不要轻视MVC模式，因为并非系统所有的部分都要被精心设计

我们的酒店基础信息重构项目，有酒店、房型、图片等多个问题子域，以房型子域为例，有房型抓取、落地及房型聚合等多个限界上下文，其中房型聚合业务逻辑简单，清晰明了，所以我们并没有采用DDD分层架构模式进行实现，而是使用MVC模式，不仅缩短工期，且可以快速上线验证效果，所以我们要避免将领域模型应用到每个限界上下文。

2、解决方案并不总是技术层面

当遇到项目问题时，我们大部分人的思维是从实现上进行解决，比如技术实现考虑高并发、高可用及高性能，但是真的有必须要吗？这让我想起一句话：业务架构是灵魂，技术架构是容器，脱离灵魂得容器是没有意义的！以我们的重构项目为例，我们在进行事件风暴前的业务用例梳理时，我们把业务进行归类，对于非核心业务我们会找相关业务使用方进行沟通确认，是否可以下线，是否可以合并，是否可以简化操作等等，最终我们在业务复杂度方面下降43%，节省大量的产研沟通和研发成本。

*   ### **非核心领域占用大量精力**
    

我们可能因为没有使用领域愿景来描述我们项目的核心竞争力，导致我们缺乏对项目成败核心是什么的关注，而我们的资源是有限的，时间是有限的，我们只有把有限的资源投入到最重要的区域-核心领域，才能更好的服务于企业战略，实现商业价值。以我们重构项目为例，我们项目开始则高亮我们的愿景，明确核心领域，把资深人员安排在核心的位置，保障核心领域的研发和交付质量。在项目最重要的区域分摊太少的资源就是反模式。

*   ### **产品不关注是否使用DDD，只关注需求是否如期上线**
    

对于产品只关注自己的需求是否如期上线，这种情况非常常见，所以需要我们要付出一些努力来达成我们做DDD重构的目标，我认为可以从以下几个方面入手：

1、讲清楚系统现状，以及使用DDD重构后所带来的价值，达成价值共识（与产品，必要可能需要跟产品的上级）；

2、资源紧张情况下，重要需求正常排期，非重要与产品沟通延迟上线；

3、优先解决系统核心价值域问题，分阶段成果发布；

4、研发资源闭环和共享模式共存。关于闭环和共享解释如下（摘自“人人都是产品经理”）：

闭环：就是和业务需求方绑定，专门做此类变化快的需求开发，其他的都不做；而共享则相反，将研发资源共享成一个池，所有的业务需求也汇总在一个或多个优先级队列里，排队开发。

共享：有利于充分利用研发资源，规模化、专业化，提升吞吐，但可能也降低了平均响应时间，更适合于进入成熟期，稳定渐进发展的业务。闭环，优先考虑专属业务需要的响应，但也失去了规模与专业化效应，更适合快速发展期的创新业务，而过了业务高速期，专属的研发就会形成资源浪费，对个体的成长也有不利因素。

**七、总结**

本文以酒店基础信息DDD重构实践为案例，介绍了我们为什么选择DDD，完整的DDD落地流程框架，以及利用大量篇幅和例子介绍我们项目落地实践的原则，最后介绍了案例成果和项目重构过程中常见的问题及解决方案，相信大家对DDD有了更加清晰的认识，同时在项目落地时也有了一些明确的指导原则，避免踩坑。但是DDD不是灵丹妙药，更不是“银弹”，最后有几点建议送给大家：

1、DDD是思想，是一种业务领域建模方法论、业务架构设计方法论，是指导开发过程的方法论

2、DDD是业务+技术的共同深度参与，开发人员需要有思考方式的转变，所以实践能否成功，不仅仅是技术的问题，更是贯彻实施的问题

3、战略设计阶段属于整个DDD核心阶段，领域边界划分对于团队的抽象能力有一定挑战，请不要为了“省事”，最后做成了“精简版”的DDD

好了，就写到这里，期待大家都能够有机会利用DDD作为项目指导和落地的方法论，并获得DDD所带来的价值体验~

![](https://mmbiz.qpic.cn/mmbiz_gif/YE1dmj1Pw7mzWKVAicdPXgb1LiaGOpovRPEO3MgG5Tlk0NEgCxNTibKeMxxVzUTXFgWibvl0LYU10HNJicib0N8jObow/640?wx_fmt=gif)

**DDD相关文章**

\# [国内酒店交易DDD应用与实践——理论篇](http://mp.weixin.qq.com/s?__biz=MzA3NDcyMTQyNQ==&mid=2649270390&idx=1&sn=ca50e0ea338ecd04d42c5b9ae422e981&chksm=87677988b010f09e6a4da49f73eb626ac26ce285fbdd78f5150ff9b37680e25e8b4a2ff77dd2&scene=21#wechat_redirect)

#

[国内酒店交易DDD应用与实践——代码篇](http://mp.weixin.qq.com/s?__biz=MzA3NDcyMTQyNQ==&mid=2649270466&idx=1&sn=cbb9be61d80b4b6c665dfe4360ecf96d&chksm=8767793cb010f02a933ece1679b705675d640fad6fbfa61cf2d941d309e6145879221628ecdf&scene=21#wechat_redirect)

#

[基于DDD思想的技术架构战略调整](http://mp.weixin.qq.com/s?__biz=MzA3NDcyMTQyNQ==&mid=2649269754&idx=1&sn=2e8f413fe3fa514013e8c0a2eb1b7d6f&chksm=87677404b010fd12d71e8eee284d69c3d122c58c54ff76a8d8fdab82b414195ae43e3157ceb7&scene=21#wechat_redirect)

#

[坚定推动DDD一年后，去哪儿网如今怎么样了](http://mp.weixin.qq.com/s?__biz=MzA3NDcyMTQyNQ==&mid=2649269734&idx=1&sn=3bf9e9f2fd244adb175ae39a6caf00f3&chksm=87677418b010fd0efd2f64b5d320f28444721a2dd19f1d5732a81694081ce716f4eab0b618c4&scene=21#wechat_redirect)

#

[听“侃王”讲基于DDD思想的酒店报价引擎重构](http://mp.weixin.qq.com/s?__biz=MzA3NDcyMTQyNQ==&mid=2649266588&idx=1&sn=bf6f020780e94d8b4a8a6de57bd02171&chksm=87674862b010c1749dd96e9ef92d0cb4af4a8dc40329325b0e01e1c7a8ffe9269d4104338832&scene=21#wechat_redirect)

#

[去哪儿网领域驱动设计（DDD）的实践之路](http://mp.weixin.qq.com/s?__biz=MzA3NDcyMTQyNQ==&mid=2649266494&idx=1&sn=137b5c95f0e5f8c430923d6d3d6d068f&chksm=876748c0b010c1d697ad1773d80ac16d0b949d280066782e26b30153095a496ccd2133abb317&scene=21#wechat_redirect)