# 实验二：房源采集与 MySQL 存储

《大数据采集与处理》｜4学时，180分钟

## 一、实验任务

采集 [安居客教学站](http://www.bspider.top/anjuke/) 的房源信息，保存为 CSV，再写入 MySQL。

本次使用 requests、BeautifulSoup 和 PyMySQL。网页解析使用 CSS 选择器。

需要采集以下8个字段：

| 字段名 | 含义 | 注意事项 |
|---|---|---|
| title | 标题 | 房源的卖点标题 |
| house | 楼盘 | 楼盘名称 |
| address | 地址 | 注意地址可能分在多个标签中 |
| struct | 户型 | 如“3室2厅1卫” |
| area | 建筑面积 | 保留面积单位 |
| floor | 楼层 | 不要取成朝向 |
| maketime | 建造年代 | 缺失时填空字符串 |
| price | 单价 | 每平方米价格，不是总价 |

**用时安排：** 环境与建表15分钟，房源解析35分钟，翻页25分钟，CSV保存20分钟，入库40分钟，结果核对25分钟，报告与提交20分钟。

> 网站说明：2026年10月26日，本次请求返回了访问验证页，未能完成在线核验。下文示例选择器已用项目保存的网页验证。实验时先检查当前页面；若出现验证页或无法访问，记录错误并联系教师。总条数以实际采集为准，不要求达到旧报告中的601条。

## 第一阶段：MySQL Docker 更新与启动

### 1. 更新课程 MySQL

已完成 L05 更新的同学直接进入下一步。尚未更新的同学，先取得教师提供的 MySQL 增量更新包。

1. 关闭正在运行的课程 MySQL。
2. 把更新包中的文件复制到原课程环境文件夹，与 `docker-compose.yml` 放在同一级。按更新包说明操作，不覆盖 `.env`、`workspace` 或原 `docker-compose.yml`。
3. 确认该目录中有 `start-mysql.bat`、`mysql-console.bat`、`l05-mysql.ps1` 和 `l05-mysql-fresh-volume.yaml`。
4. 保留原数据卷。若提示卷或配置异常，请教师检查。

更新使用独立的 L05 教学数据卷。以后上课继续使用这个卷，不需要每次更新。

### 2. 依次打开三个脚本

先打开 Docker Desktop，等引擎启动。再进入原课程环境文件夹，依次操作：

| 顺序 | 双击文件 | 用途 | 成功标志 |
|---|---|---|---|
| 1 | `start-course.bat` | 启动课程环境 | 浏览器打开 JupyterLab |
| 2 | `start-mysql.bat` | 启动课程 MySQL | 显示 `[PASS] L05 MySQL healthy，账号连接检查通过。` |
| 3 | `mysql-console.bat` | 打开 MySQL 终端 | 输入密码后出现 `mysql>` |

第三步出现 `Enter password:` 时，输入课程数据库密码 `123456`，按回车。终端输入密码时不显示字符，这是正常现象。

如果缺少脚本，向教师获取 MySQL 更新包。脚本报错时保留提示，请教师检查，不要自行删卷或重装。

脚本会使用已有的 `docker-compose.yml` 和 `l05-mysql-fresh-volume.yaml`，复用 L05 教学数据卷。学生无需手写 Docker 启动命令。

### 3. 准备实验文件

在实验一的原作业仓库中，新建以下目录：

```text
lab02/
  anjuke.ipynb
  data/
  images/
  REPORT.md
```

在 JupyterLab 中新建 `anjuke.ipynb`。也可以自行使用 `anjuke.py`。

若缺少依赖，在 Notebook 单元中运行：

```python
%pip install requests beautifulsoup4 pymysql
```

BeautifulSoup 的安装名是 `beautifulsoup4`，导入名是 `bs4`。

### 4. 建立房源表

以下 SQL 都在 **MySQL 终端**中执行。每条语句末尾加分号，不要复制 `mysql>` 提示符。

#### 查看数据库和数据表

```sql
SHOW DATABASES;
USE smartpark_lab;
SHOW TABLES;
```

`SHOW DATABASES` 查看数据库；`USE` 切换数据库；`SHOW TABLES` 查看当前库中的表。

如果没有 `smartpark_lab`，请教师检查环境。

#### 创建 houseinfo 表

表不存在时，执行下面的完整 SQL：

```sql
CREATE TABLE IF NOT EXISTS houseinfo (
    id INT AUTO_INCREMENT PRIMARY KEY COMMENT '自增长主键',
    title VARCHAR(200) COMMENT '标题',
    house VARCHAR(50) COMMENT '楼盘',
    address VARCHAR(200) COMMENT '地址',
    struct VARCHAR(50) COMMENT '户型',
    area VARCHAR(20) COMMENT '建筑面积',
    floor VARCHAR(20) COMMENT '楼层',
    maketime VARCHAR(20) COMMENT '建造年代',
    price VARCHAR(20) COMMENT '每平方米单价'
) CHARACTER SET utf8mb4;
```

`id` 由数据库自动生成。其余8列保存采集结果，暂时保留面积、价格中的单位。

#### 检查表结构，记录入库前条数

```sql
DESCRIBE houseinfo;
SELECT COUNT(*) AS before_count FROM houseinfo;
```

保存表结构截图，记下 `before_count`，后面用来计算新增条数。

## 第二阶段：房源采集与 CSV 数据存储

本阶段完成网页采集和 CSV 保存，暂不向数据库写入数据。

### 1. 获取第一页

在 Notebook 中请求教学站，检查返回内容：

```python
import requests
from bs4 import BeautifulSoup

url = "http://www.bspider.top/anjuke/"
r = requests.get(url, headers={"User-Agent": "Mozilla/5.0"}, timeout=30)
r.raise_for_status()
r.encoding = r.apparent_encoding
soup = BeautifulSoup(r.text, "html.parser")
```

这段代码完成三件事：请求网页、设置编码、生成 BeautifulSoup 对象。`timeout=30` 表示超时后报错；`raise_for_status()` 用来检查 HTTP 错误。

**自己检查：** 输出状态码和网页标题，确认内容是房源列表。即使状态码为200，也可能是访问验证页。遇到验证页或网络错误，先停止采集并记录，不要改采真实安居客网站。

#### 网站无法采集时

教师确认使用网页快照后，把教师提供的 HTML 文件放入 `lab02/html/`。只需网页文件，不领取教师源码或CSV答案。用 `Path.read_text(encoding="utf-8")` 读取HTML，再交给BeautifulSoup，继续完成字段解析、CSV和入库。

分页部分仍要分析快照中的 `href`、`urljoin` 和末页按钮，但按文件读取属于离线回放，不能写“在线翻页成功”。报告中注明快照文件名；无法完成的浏览器操作写明原因，用快照中的HTML节点作为补充证据。

### 2. 分析网页，提取8个字段

#### 复制一个字段的选择器

1. 用 Chrome 打开教学站，按 **F12**。
2. 切到 **Elements（元素）**。
3. 点击左上角的选取元素按钮，再点击第一条房源的标题。
4. 在选中的 HTML 节点上右键，选择 **Copy → Copy selector**。
5. 把复制结果记入报告。

复制结果往往很长，可能包含多层 `>` 和 `:nth-child()`。例如：

`#页面根节点 > div:nth-child(3) > … > h3`

这只是结构示意，实际值要从自己的页面复制。页面多插入一个元素，按位置选择的路径就可能失效。因此还需要找到更稳定的选择方法。

#### 找到“一条房源”的父容器

从标题节点向上查看父节点，把鼠标移到节点上，观察网页高亮区域。

找到一个**刚好包住整条房源**的节点。它应同时包含标题、楼盘、地址、属性和价格。再检查第二条房源，看看它们是否使用相同的 class。

已有网页中的房源容器是 `div.property`。先在当前页面核对，再运行：

```python
rows = soup.select("div.property")
print(len(rows))
```

`select()` 返回所有匹配节点，结果是列表。这里列表中的每一项对应一条房源。若结果为0，先检查网页和选择器。

#### 在一条房源内找标题

确认 `rows` 不为空后，取第一条房源：

```python
row = rows[0]
node = row.select_one("h3")
```

`select_one()` 返回第一个匹配节点；找不到时返回 `None`。先检查节点是否存在，再用 `node.get_text(strip=True)` 读取文字。

提取地址等字段时，不要在相邻子标签之间主动插入空格；保留文字中原有的分隔。CSV保存后，MySQL直接使用CSV中的文字，不另改空格格式。

这里从 `row` 内部查找 `h3`，范围只限于这一条房源。这就是相对选择器。处理下一条房源时，使用同样的选择方法。

**提炼选择器的思路：** 先选中一条房源，再在里面找字段，不必保留从整个页面根节点开始的长路径。

在 Elements 面板按 **Ctrl+F** 输入选择器，可以查看匹配结果。最后还要在 Python 中运行一次，确认请求得到的 HTML 也能匹配。

#### 自己定位其余字段

按同样的方法分析楼盘、地址、户型、面积、楼层、年代和单价。把每个字段的选择器和取值方法记入报告。

重点检查：

- **地址、户型：** 文字可能分散在多个子标签中，要取完整文字。
- **面积、楼层：** 先打印属性组中的每项内容，确认顺序，不能把朝向当成楼层。
- **建造年代：** 先判断节点或列表项是否存在，缺失时填空字符串。
- **单价：** 对照网页中的单位，不能取成以“万元”为单位的总价。

如果用列表下标取值，必须先检查长度。例如，取第5项前要确认长度大于4。只检查“列表非空”不能防止下标越界。

**本步任务：** 自己编写解析函数，按下列顺序返回一条记录：

`title → house → address → struct → area → floor → maketime → price`

先解析第一条，逐字段对照网页；再检查第二条和最后一条。确认正确后，再处理整页。面积或单价缺失时，先查原因，不要直接跳过。

**截图：** 保留选择器推导过程；框出 `select`、`select_one`，以及面积和单价的定位代码。

### 3. 采集下一页

#### 检查“下一页”链接

在网页底部右键“下一页”，选择“检查”。记录该节点的 class 和 `href`。

已有网页中可用 `a.next.next-active` 找到活动的下一页按钮。核对当前页面后，用 `select_one()` 定位，用 `.get("href")` 读取链接。

注意：`get_text()` 取得的是“下一页”文字，`get("href")` 取得的才是地址。

#### 补全链接

下一页链接可能只有 `?page=2`，需要用当前页面地址补全：

```python
from urllib.parse import urljoin
next_url = urljoin(current_url, href)
```

先输出 `href` 和 `next_url`，再打开合成的地址，检查是否确实进入下一页。

#### 自己编写分页循环

按以下顺序组织程序：

1. 请求当前页，检查响应内容。
2. 解析本页房源，加入结果列表。
3. 找下一页，补全地址。
4. 检查地址没有越出教学站，且没有访问过。
5. 等待至少1秒，再请求下一页。

先试采2页，确认第二页没有重复第一页，再继续采集。

每页输出页码、URL、本页条数和累计条数。只有确认分页区域存在，且“下一页”按钮已禁用，才能判断到达末页。整个分页区域消失，或找不到按钮，应记为结构异常，不能算采集完成。

遇到请求失败、空列表、重复地址或重复页面，应报错并记录。设置一个安全页数上限，防止无限循环；达到上限只能记为中止，不能算采集完成。

**截图：** 框出下一页 `href`、`urljoin`、循环更新地址和停止条件，并附实际翻页日志。

### 4. 保存和核对 CSV

自己编写 CSV 保存代码，文件名为 `data/houseinfo.csv`。

| 要点 | 要求 |
|---|---|
| 写入工具 | 使用 `csv.writer` |
| 文件参数 | 编码 `utf-8-sig`，`newline=""` |
| 表头 | 使用前面的8个字段名，顺序一致 |
| 数据 | 每条记录恰好8项 |
| 重复记录 | 按8个字段完全相同去重后保存，并记录去重前后条数 |

用 `csv.DictReader` 重新读取文件，检查数据行数，表头不计入数据行数。

选第一条、跨页一条和最后一条，与网页逐字段核对。重点检查地址是否完整、面积和单价单位是否正确。

**截图：** 框出 CSV 保存和回读代码，展示回读条数及样本核对结果。

## 第三阶段：MySQL 数据存储

**本阶段读取第二阶段保存的 `data/houseinfo.csv`，把其中的数据写入 MySQL，不重新爬取网页。**

CSV与数据库中的8个房源字段应一致，顺序、文字和单位保持不变。数据库额外生成 `id`。本实验新建房源表，首次入库后的条数应等于去重后的CSV数据行数。

这一部分由你独立编写。先实现数据库连接，再完成批量写入。

### 1. 连接信息从哪里来

Python 程序在课程 JupyterLab 中运行。连接目标如下：

| 连接参数 | 说明 |
|---|---|
| host | `mysql`，课程数据库服务名 |
| port | `3306`，整数 |
| database | `smartpark_lab` |
| user | `student` |
| password | `123456`，课程 Docker 教学数据库密码 |
| charset | `utf8mb4` |
| autocommit | 设为 `False`，由程序提交事务 |

连接时直接使用上表中的课程教学参数，无需修改 Docker 密码。

### 2. 重点函数

**① `pymysql.connect()`：连接数据库**

传入上表中的连接参数，返回连接对象。后面的游标、提交和回滚都通过这个连接完成。

**② `连接.cursor()`：创建游标**

返回游标对象。游标负责执行 SQL、获取查询结果。

**③ `游标.execute(sql, params)`：执行一条 SQL**

`sql` 是 SQL 语句，`params` 是这条语句所需的参数。返回值是受影响行数。执行查询后，要用 `fetchone()` 或 `fetchall()` 取回数据。

**④ `游标.executemany(sql, records)`：批量执行 SQL**

`records` 是多条记录组成的列表，每条记录要有8个值，顺序与 INSERT 中的字段一致。返回受影响行数，不是数据库总条数。

SQL 中用 `%s` 表示参数位置，实际值通过第二个参数传入。不要用字符串拼接或 f-string 把数据塞进 SQL。

**⑤ `连接.commit()`：提交**

写入成功后调用。提交后，才能在另一个 MySQL 终端中核对新数据。

**⑥ `连接.rollback()`：回滚**

写入出错时调用，撤销本次事务中尚未提交的修改。它不能撤销已经提交的数据，也不能撤销 MySQL 建表操作。

**⑦ `游标.close()`、`连接.close()`：关闭**

结束时先关闭游标，再关闭连接。成功或失败都应释放资源。连接失败时，对象可能尚未创建，要先判断再回滚或关闭。

### 3. 独立完成入库程序

请完成以下任务：

1. 用 `csv.DictReader` 读取第二阶段的CSV，检查表头，并按8个字段的固定顺序整理记录。
2. 连接数据库，查询一条数据，确认连接正常。
3. 编写 INSERT，写清8个字段和8个参数位置。
4. 用 `executemany()` 写入多条记录，成功后 `commit()`。
5. 出错时 `rollback()`，最后关闭游标和连接。
6. 先用少量真实记录检查，再写入全部记录。
7. 处理重复运行：比较库内已有记录，避免把同一批数据再插入一次。

自增长 `id` 不能防止房源重复。课堂单人运行时，可按8个字段比较已有记录，只插入尚不存在的记录。不要通过清空表来完成重跑。

**截图：** 框出自己编写的 `executemany()`、`commit()`、`rollback()` 和关闭操作，解释参数如何对应8个字段。

### 4. 在 MySQL 终端验收入库结果

重新双击 `mysql-console.bat`，输入密码后执行：

```sql
SHOW DATABASES;
USE smartpark_lab;
SHOW TABLES;
DESCRIBE houseinfo;
SELECT COUNT(*) AS after_count FROM houseinfo;
SELECT title, house, address, struct, area, floor, maketime, price
FROM houseinfo ORDER BY id LIMIT 5;
```

记录入库前后条数，计算本次新增条数。再运行同一批数据一次，重新查询，检查是否重复增加。

抽取 CSV 中的3条记录，在数据库中逐字段核对，确认内容相同。

**截图：** 保留 `DESCRIBE`、入库前后及重跑后的 `COUNT`、样本查询结果。未完成的步骤如实写“未完成”。

## 报告与提交

使用 `REPORT.md` 模板。每张代码截图只截相关部分，显示文件名、行号或单元位置，用红框标出关键代码，并写2—3句话解释。

必须保留以下证据：

- 选择器推导，以及 `select`、`select_one` 的使用。
- 面积、单价和年代的解析。
- 下一页链接、地址补全、循环和停止条件。
- CSV保存与核对。
- 批量入库、提交、回滚和关闭。
- MySQL表结构、条数和样本核对。

仅截最终结果，或只贴 AI 生成的文字，不能代替个人实验过程。

### 1. 整理要提交的文件

先保存 Notebook 和报告。如果文件只在 JupyterLab 中，右键文件，选择 **Download（下载）**，保存到电脑。截图也放到下面对应的目录中：

```text
lab02/
  anjuke.ipynb（或 anjuke.py）
  REPORT.md
  data/
    houseinfo.csv
  images/
    你的截图文件.png
```

报告中的图片路径要与文件名一致。例如，截图放在 `images/area-price.png`，报告中就写 `![面积与单价](images/area-price.png)`。

### 2. 打开实验一的原仓库

1. 登录 GitHub，打开实验一提交过的仓库。
2. 点击 **Code**，回到仓库文件列表。
3. 确认能看到实验一的文件。**本次继续用这个仓库，不新建仓库，也不覆盖实验一文件。**

### 3. 在网页中新建 lab02 文件夹

GitHub 通过创建文件来建立文件夹：

1. 在仓库首页点击 **Add file → Create new file**。
2. 在文件名输入框中输入 `lab02/REPORT.md`。输入 `/` 后，前面的 `lab02` 就成为文件夹名。
3. 将自己已经填写好的报告内容粘贴到编辑区。
4. 点击 **Commit changes…**，填写说明，例如“添加实验二报告”。
5. 选择直接提交到当前默认分支，再点击 **Commit changes** 确认。
6. 返回仓库首页，点击 `lab02`，应能看到 `REPORT.md`。

如果已经有 `lab02`，直接打开它。已有 `REPORT.md` 时，点击文件，再点击铅笔按钮修改，不必重复创建。

### 4. 上传源码、CSV和截图

1. **进入仓库中的 `lab02` 文件夹**，点击 **Add file → Upload files**。
2. 从电脑中把源码文件，以及整个 `data`、`images` 文件夹拖入上传区域。这里拖入的是 `lab02` 里面的内容，不是外层 `lab02` 文件夹。
3. 检查待上传路径，应包括 `anjuke.ipynb`、`data/houseinfo.csv` 和 `images/截图文件名.png`。不要变成 `lab02/lab02/`，也不要把截图全放到仓库首页。
4. 填写提交说明，例如“上传实验二源码、CSV和截图”，点击 **Commit changes**。

若只上传单个文件，也可以点击 **choose your files** 选择。CSV要在网页的 `lab02/data` 目录中上传，截图要在 `lab02/images` 中上传，避免丢失目录层级。

上传前检查文件中没有个人账号密码、`.env` 或教师参考答案。课程教学数据库使用本手册给出的统一密码。看到文件出现在仓库列表中，才算提交完成。

### 5. 检查后提交学习通

1. 在 GitHub 打开源码，确认是本次保存的版本。
2. 打开 `data/houseinfo.csv`，确认有表头和数据。
3. 打开 `REPORT.md`，确认文字完整、图片能显示。图片不显示时，核对路径、大小写和扩展名。
4. 返回仓库首页，复制浏览器地址栏中的**仓库链接**。
5. 将该链接提交到学习通。仍交实验一的原仓库链接，不交本机路径或单张截图链接。

网页操作可参考 GitHub 官方说明：[创建文件](https://docs.github.com/en/repositories/working-with-files/managing-files/creating-new-files)、[上传文件](https://docs.github.com/en/repositories/working-with-files/managing-files/adding-a-file-to-a-repository)。

### 思考题

1. 下一页是怎样取得的？结合自己的代码，解释 href、urljoin 和停止条件。
2. 旧代码把 out_list 定义在全局。你把结果列表放在哪里？为什么？
3. 你怎样获取建造年代？节点缺失或顺序改变时，怎样处理？
