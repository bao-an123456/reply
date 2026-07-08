## 1. queryList(ReimbursementQueryDTO queryDTO)

1. **创建分页参数**：使用 `queryDTO` 传入的 `pageNum` 和 `pageSize` 初始化 MyBatis-Plus 的 `Page<FkReimMain>` 对象。
2. **构建查询条件**：实例化 `LambdaQueryWrapper<FkReimMain>` 条件构造器，并根据 `queryDTO` 的入参进行条件拼接：
   - 模糊匹配 (`like`)：单号 `reimNo`、标题 `title`、出差原因 `reason`。
   - 等值匹配 (`eq`)：公司 ID `companyId`、部门 ID `deptId`、报销人 ID `reimburserId`、业务类型 ID `businessTypeId`。
3. **设置排序**：按创建时间倒序排列（`orderByDesc(FkReimMain::getCreationTime)`）。
4. **执行分页查询**：调用 MyBatis-Plus 的 `this.page(page, wrapper)` 查询符合条件的数据。
5. **转换并返回**：遍历查询出的 `FkReimMain` 列表，将每个实体的字段值逐一映射赋值给 `ReimbursementListVO`，格式化更新时间与创建时间为 `yyyy-MM-dd HH:mm:ss`，最后封装入新的分页对象中返回。

---

## 2. getDetail(String id)

1. **查询主表**：调用 `this.getById(id)` 获取报销单主表实体，若为空则直接返回 `null`。
2. **查询行程**：调用 `tripService.getTripsByReimId(id)` 查出关联的所有行程，并将行程列表转为以行程 ID 为 Key 的 Map 结构。
3. **查询补助与日历明细**：
   - 调用 `subsidyService.getSubsidiesByReimId(id)` 查出所有关联的补助。
   - 遍历补助列表，根据补助项里的 `tripId` 从行程 Map 中获取到达城市编号（`arrivalCityNo`）赋值给前端对应的补助城市字段。
   - 针对每条补助，调用 `calendarService.getCalendarsBySubsidyId` 查询对应的每日补助勾选及标准明细。
4. **查询费用分摊**：调用 `costAllocationService.getAllocationsByReimId(id)` 获取分摊数据列表。
5. **组装返回**：将主表数据、行程 VO 列表、补助（含日历）VO 列表、费用分摊 VO 列表统一组装进 `ReimbursementDetailVO` 对象并返回。

---

## 3. saveDraft(ReimbursementSaveDTO saveDTO)

1. **调用保存引擎**：直接调用类私有核心保存方法 `saveOrUpdateMain(saveDTO, "0")`，传入状态值 `"0"`（表示草稿）。
2. **返回结果**：返回生成或已存在的报销单主键 ID。

---

## 4. submitReimbursement(ReimbursementSaveDTO saveDTO)

1. **表单非空校验**：校验 `saveDTO` 中标题、报销人 ID、部门 ID、公司 ID、业务类型 ID 是否为空，若空则抛出 `RuntimeException`。
2. **从表存在校验**：校验行程明细列表、费用分摊列表是否为空，若空则抛出 `RuntimeException`。
3. **分摊比例和校验**：遍历 `saveDTO` 里的分摊列表，累加所有分摊项的分摊比例，校验最终比例和是否等于 100%（`BigDecimal.ONE`），不等于则抛出异常。
4. **分摊金额和校验**：累加所有分摊项的分摊金额，校验最终金额和是否与补助总金额相等，不相等则抛出异常。
5. **调用保存引擎**：校验全部通过后，调用 `saveOrUpdateMain(saveDTO, "1")`，传入状态值 `"1"`（表示已提交/待审批）。

---

## 5. deleteReimbursement(String id)

1. **删除行程**：调用 `tripService.deleteTripsByReimId(id)` 删除与该报销单关联的所有行程明细。
2. **删除补助与日历**：调用 `subsidyService.deleteSubsidiesByReimId(id)`，其内部会同步执行 `calendarService.deleteCalendarsBySubsidyId` 删除补助信息和日历明细。
3. **删除费用分摊**：调用 `costAllocationService.deleteAllocationsByReimId(id)` 删除关联的分摊数据。
4. **删除主表**：调用 `this.removeById(id)` 删除报销单主表记录。

---

## 6. cancelReimbursement(String id)

1. **获取主表实体**：调用 `this.getById(id)` 获取报销单主表记录。
2. **更新状态**：若获取的实体不为空，将其 `status` 字段修改为 `"2"`（表示已作废）。
3. **重置更新时间**：设置 `updateTime` 为当前时间 `LocalDateTime.now()`。
4. **执行更新**：调用 `this.updateById(main)` 将修改保存至数据库。

---

## 7. saveOrUpdateMain(ReimbursementSaveDTO saveDTO, String status) 【核心私有方法】（保存或更新报销单主表以及所有关联的明细表）

1. **判定新增与修改**：
   - 检查传入的 ID 是否为空或包含临时标记（如 `"NEW_"`、`"COPY_"` 等）。
   - 若为**修改**，先根据该 ID 查询出原有报销单，并保留其原单号和原创建时间。
   - 若为**新增**，将主表实体 ID 设为 `null` 并将创建时间设为当前时间；根据当天日期格式化出单号前缀（如 `RCBX20260708`），模糊统计数据库内当天已存在单号的数量后 `+1`，最终格式化拼接为当天的唯一单号。
2. **补全主表冗余快照字段**：
   - 根据报销人 ID 去基础员工表中查出其工号和姓名写入主表。
   - 同理根据部门 ID、公司 ID 和业务类型 ID，分别去基础部门表、基础公司表和基础业务类型表中查出对应的编号与名称冗余写入主表。
3. **设置主表基本字段**：
   - 填充标题、出差原因、餐补/交通补/通讯补总计金额、补助总额、备注、状态（`status`）及更新时间。
4. **落库主表**：
   - 调用 `this.saveOrUpdate(main)`。若属于新增，主键 ID 会被自动生成并写回主表实体（后续称 `reimId`）。
5. **清理旧从表数据（仅修改时）**：
   - 若是修改操作（`!isNew`），为保证数据一致，采用“先删后插”策略，物理清空数据库中该报销单所有的行程明细、补助明细、补助日历明细以及费用分摊明细。
6. **批量保存新行程数据并替换临时 ID**：
   - 循环遍历最新的行程列表：
     - 若为新增明细项则将 ID 设置为 `null`；若为已有明细项则保留 ID。
     - 分别根据出行人 ID、出发城市编号、到达城市编号，去基础表中查询出行人的姓名和工号、城市名称补全到行程记录中。
     - 填充排序号 `sortOrder`，加入集合。
     - 调用 `tripService.saveBatch(tripEntities)` 批量写入。
     - **映射替换**：由于保存后数据库生成了真正的行程 ID，代码利用数组索引的对应关系，遍历将补助明细列表 `subsidies` 中关联的前端临时行程 ID 替换为刚才落库生成的最新的真实行程 ID。
7. **批量保存补助明细及日历明细**：
   - 校验补助列表不为空，级联调用 `subsidyService.saveSubsidiesAndCalendars(reimId, saveDTO.getSubsidies(), isNew)` 保存。
8. **批量保存费用分摊明细**：
   - 循环遍历费用分摊明细列表：
     - 判定明细 ID，新增项设为 `null`。
     - 根据分摊的公司 ID、项目 ID 分别去基础公司表、基础项目表中查询对应的名称与编号并补全。
     - 设置排序号 `sortOrder` 后加入集合。
     - 调用 `costAllocationService.saveBatch(allocEntities)` 批量写入。
9. **返回主表 ID**：
   - 返回已入库/更新的主单 ID。
