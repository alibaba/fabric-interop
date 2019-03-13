# cmcc方案的问题
1. cmcc链码发布、版本管理问题。
2. cmcc链码在通道中各个不同组织的初始化流程。
3. cmcc的实例化策略和背书策略太固化。
	* 实例化策略固化使得整个联盟的管理显得有一些中心化。
	* 背书策略固化使得后加入的组织无法参与背书，背书风险集中。
4. 太多工作只能在链码外完成。
	* configupdate的计算
	* 当前proposal是否已经收集到足够的签名
	* 当前propoal是否因为有其他configupdate的提交而已经失效

# cmscc方案的优势
1. 链码内置在源码中，由社区发布。
2. 链码无需初始化。
3. 链码无需实例化策略。背书策略可以在vscc中实现，可以设置为SignedByMajorityPeerorgs。新加入的组织可以自动参与背书。
4. 链码能够获取最新的configblock，之前必须在链码外完成的工作现在可以在链码中完成。我们可以进一步把互操作流程中的很多逻辑固化到链码中，能更好的规范互操作的流程。

# POC场景说明
当前存在一个由Org1和Org2两个组织构成的通道。在几乎同一时间，OrgX经由Org1介绍加入通道；而OrgY经由Org2介绍加入通道。

1. Org1提交proposal "add_orgx"
2. Org1为proposal "add_orgx" 签名，并提交签名
3. Org2提交proposal "add_orgy"
4. Org2为proposal "add_orgy" 签名，并提交签名
5. Org2发现proposal "add_orgx"，为它签名，并提交签名
6. Proposal "add_orgx"签名已经集齐，cmscc通过模拟configupdate的方式，验证这个proposal已经是可以提交状态，会产生config_update_envelope
7. Org2或Org1提交proposal "add_orgx"中的config_update_envelope
8. "add_orgx"的config_update_envelope成功执行，OrgX加入通道。
9. 此时，proposal "add_orgx"的状态变为"Submitted"，而proposal "add_orgy"的状态变为"Outofdate"。因为config已经升级，config里"Application"字段的版本改变，proposal "add_orgy"的config_update需要重新计算。
10. Org2调用UpdateProposals，该接口会清除已经提交的proposal，重新计算config_update
11. 重新收集proposal "add_orgy"的签名（略……）

# 技术细节
## 智能合约存储什么？
存储所有的proposal，每个proposal包含以下内容：

1. config是序列化后的Config结构。存储原始信息。
	* 其中Sequence存储计算ConfigUpdate时当前configblock的sequence
	* 其中ChannelGroup存储用户的输入，他是一种"patch"的形式，存储用户的原始意图。（例如增加组织，就是在Application.Groups里增加一条KV）
2. config_update是序列化后的ConfigUpdate结构，存储待签名的ConfigUpdate。
3. signatures的value是序列化后的ConfigSignature结构。存储各个组织上传的签名。

## ProposeConfigUpdate流程（AddOrgnization、ReconfigOrgnizationMSP与该流程逻辑类似）
![image](http://gitlab.alibaba-inc.com/aliyun-blockchain/fabric-interop/raw/master/imgs/propose_config_update.png)

## GetProposal流程（ListProposals、UpdateProposals与该流程逻辑类似）
![image](http://gitlab.alibaba-inc.com/aliyun-blockchain/fabric-interop/raw/master/imgs/get_proposal.png)

# FAQ
Q:为何cmscc不主动发起config update的交易，而是需要org1或者org2发起？
A:设计上智能合约应该是“被动的”，智能合约能主动发起交易是很奇怪的行为。技术上，peer节点不是ChannelWriter，也是无法提交交易给orderer的。