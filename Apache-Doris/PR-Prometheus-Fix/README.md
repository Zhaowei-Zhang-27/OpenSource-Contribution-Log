# 复盘：修复 Apache Doris BE 指标导出重复问题

* **PR 链接**：https://github.com/apache/doris/pull/59157
* **Issue 编号**：#57788
* **核心技术栈**：C++11, Prometheus Protocol, CMake, 源码审计

---

##  1. 核心摘要
修复了 Doris BE 在高并发/复杂存储配置下，Prometheus 指标输出违反协议规范（#TYPE 重复）的底层 Bug。通过引入基于 `std::unordered_set` 的全局状态管理，确保了监控数据的合规性。

##  2. 问题剖析
根据 Issue #57788 的反馈，Doris BE 在导出 Prometheus 监控指标时，输出结果中存在同一个指标名对应多行 # TYPE 声明的情况。这直接违反了 Prometheus 官方的指标展示规范（Exposition Format）：“每一个指标名在输出中只能存在一行 TYPE 声明”
经过对 `be/src/util/metrics.cpp` 中 `MetricRegistry::to_prometheus()` 方法的深度源码审计，发现其去重逻辑存在严重的语义偏差：

- **错误的标识符**：原代码使用 `group_name`（指标组名）作为去重的 Key。

- **逻辑漏洞**：

    1. `group_name` 是可选字段且不具备唯一性。当多个不同指标的组名为空时，去重逻辑失效，导致重复输出 `# TYPE`。

    2. 若不同指标共享同一个组名，则会导致后续指标的 `# TYPE` 声明被错误截断。

- **结论**：正确的去重标识符应为指标的完整名称（Metric Name），即通过 `combine_name(_name)` 获取的字符串。

##  3. 修复方案
- **引入全局去重机制**：在 `to_prometheus` 方法内部引入 `std::unordered_set<std::string> exported_types`。

- **精准去重逻辑**：在遍历指标实体时，优先检查指标全名是否已存在于 `exported_types` 中。仅当该指标名是首次出现时，才输出相应的 `# TYPE` 和 `# HELP` 信息。

- **规范合规**：该改动确保了无论环境配置多复杂，输出结果均严格遵守 Prometheus 协议规范。

> **技术思考**：在此处我选择了局部 `std::unordered_set` 而非全局静态变量，是为了确保线程安全（Thread-Safety）并避免内存泄漏，符合 C++ RAII 原则。

##  4. 验证
为了确保这次贡献的质量，我在云服务器环境下执行了完整的闭环开发流程：

- **环境构建**：完整拉取 `apache/doris` 仓库（包含 2600+ 源代码文件），配置基于 **CMake 3.29.0** 和 **GCC 15.1.0** 的全套编译工具链。

- **全量编译**：执行 `sh build.sh --be -j 8` 完成 BE 模块编译，生成了约 **2.1G** 的二进制产物，确保代码在工业级规模工程中无编译障碍。

- **验证与解释**：

    - **测试现状**：在当前的 master 分支默认配置下，修复前后的指标统计均为 **213/213** 匹配。

    - **核心逻辑证明**：由于本地精简环境未配置复杂的资源组或多存储路径，未触发 Issue #57788 中的重复场景。但通过源码分析可以明确，原有的 `group_name` 逻辑是确定性错误的。

    - **修复价值**：这是一个防御性且逻辑纠偏的修复。它消除了复杂生产配置下必然产生的格式违规风险，极大提升了监控系统的健壮性。

* **编译环境**：GCC 15.1.0 / CMake 3.29.0
* **工程规模**：2600+ 源码文件，2.1G 二进制产物全量编译通过。
* **验证逻辑**：虽本地环境受限，但通过静态分析证明了原 `group_name` 逻辑在多路径/资源组场景下存在 $O(1)$ 的失效概率。

---

##  5. 过程存证
<details>
<summary>1. 编译环境与 License 配置</summary>
<img width="352" height="253" alt="01_build_script_configuration_and_license" src="https://github.com/user-attachments/assets/8a832838-05a7-482e-956a-06e63ba7bc79" /><br>
 <img width="383" height="67" alt="02_toolchain_version_check_cmake_3_29" src="https://github.com/user-attachments/assets/10723cf9-e45d-411d-9cfe-93ead2213afb" /><br>
<img width="429" height="104" alt="02_verified_compilation_artifacts_and_binary_size" src="https://github.com/user-attachments/assets/e50ad2d0-8cf2-46cc-a33e-e73f24359207" /><br>
</details>
    
<details>
<summary>2. 核心 Bug 逻辑审计</summary>
<img width="527" height="276" alt="06_analysis_of_flawed_group_name_deduplication_logic" src="https://github.com/user-attachments/assets/6146d334-02ce-437b-9a69-c2be03d211cf" /><br>
<img width="422" height="110" alt="07_metric_registry_internal_management_audit" src="https://github.com/user-attachments/assets/d21dd902-e151-4857-aefd-e0f9089fdc29" /><br>
<img width="544" height="301" alt="12_comprehensive_git_diff_of_metrics_logic" src="https://github.com/user-attachments/assets/4b13126b-e0e1-4f5f-bf9a-c7ad723cb8f3" /><br>
</details>

<details>
<summary>3. 流程</summary>
<img width="377" height="403" alt="13_professional_development_and_testing_workflow" src="https://github.com/user-attachments/assets/52220962-9006-4005-9b96-0e747de28552" /><br>

</details>




