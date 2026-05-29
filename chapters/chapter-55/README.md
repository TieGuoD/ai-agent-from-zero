# 第55章：Agent 与文件系统 —— 文件操作 Agent

## 学习目标

通过本章的学习，你将能够：
1. 理解文件操作 Agent 的设计原理
2. 掌握安全的文件读写操作方法
3. 学会构建支持多种文件格式的处理系统
4. 理解文件组织和管理的自动化
5. 掌握文件搜索和内容提取技术
6. 能够构建完整的文件管理 Agent

## 核心问题

文件是计算机中最基本的数据存储单元。每天，我们都在进行大量的文件操作：创建、编辑、移动、删除、搜索、整理。这些操作虽然简单，但重复执行会消耗大量时间和精力。

文件操作 Agent 可以帮助我们自动化这些任务：
1. 理解文件内容和结构
2. 智能地组织和分类文件
3. 批量处理多个文件
4. 搜索和检索文件内容
5. 自动备份和同步

这就像是给文件系统配了一个智能管家。

---

## 原理讲解

### 文件操作 Agent 的核心能力

一个完整的文件操作 Agent 应该具备以下能力：

**文件读写**：安全地读取和写入各种格式的文件。

**内容理解**：理解文件的内容和结构。

**文件组织**：自动分类、重命名和整理文件。

**批量处理**：高效地处理大量文件。

**搜索检索**：快速找到需要的文件和内容。

### 安全考虑

文件操作是敏感操作，需要特别注意安全：

**权限控制**：只访问授权的目录和文件。

**操作审计**：记录所有的文件操作。

**备份机制**：重要操作前自动备份。

**沙箱执行**：在隔离环境中处理不可信的文件。

### 文件格式支持

文件操作 Agent 需要支持多种文件格式：

**文本文件**：TXT、CSV、JSON、XML 等。

**文档文件**：PDF、Word、Excel 等。

**媒体文件**：图片、音频、视频等。

**压缩文件**：ZIP、TAR、RAR 等。

### 智能文件管理

Agent 可以实现智能的文件管理：

**自动分类**：根据内容自动分类文件。

**重复检测**：识别重复的文件。

**版本管理**：管理文件的历史版本。

**标签系统**：为文件添加标签便于检索。

---

## 完整代码示例

### 示例 1：基础文件操作 Agent

```python
"""
基础文件操作 Agent
安全地执行文件操作
"""

import os
import shutil
import json
import hashlib
from pathlib import Path
from typing import Dict, List, Optional, Tuple
from dataclasses import dataclass, field
from datetime import datetime
import anthropic


@dataclass
class FileOperation:
    """文件操作记录"""
    operation_type: str
    source_path: str
    destination_path: Optional[str] = None
    success: bool = True
    message: str = ""
    timestamp: str = field(default_factory=lambda: datetime.now().isoformat())


class FileSystemAgent:
    """文件系统 Agent"""
    
    def __init__(self, base_dir: str = "."):
        self.base_dir = Path(base_dir).resolve()
        self.client = anthropic.Anthropic(
            api_key=os.environ.get("ANTHROPIC_API_KEY")
        )
        self.operations: List[FileOperation] = []
        self.allowed_directories: List[Path] = [self.base_dir]
    
    def _is_allowed_path(self, path: Path) -> bool:
        """检查路径是否允许"""
        try:
            resolved = path.resolve()
            return any(
                str(resolved).startswith(str(allowed))
                for allowed in self.allowed_directories
            )
        except:
            return False
    
    def read_file(self, file_path: str) -> str:
        """读取文件"""
        path = Path(file_path)
        
        if not self._is_allowed_path(path):
            return "错误: 无权访问该路径"
        
        try:
            with open(path, 'r', encoding='utf-8') as f:
                content = f.read()
            
            self.operations.append(FileOperation(
                operation_type="read",
                source_path=str(path),
                success=True,
                message=f"读取成功，{len(content)} 字符",
            ))
            
            return content
            
        except Exception as e:
            return f"读取失败: {e}"
    
    def write_file(self, file_path: str, content: str, 
                   mode: str = "w") -> bool:
        """写入文件"""
        path = Path(file_path)
        
        if not self._is_allowed_path(path):
            print("错误: 无权访问该路径")
            return False
        
        try:
            # 确保目录存在
            path.parent.mkdir(parents=True, exist_ok=True)
            
            with open(path, mode, encoding='utf-8') as f:
                f.write(content)
            
            self.operations.append(FileOperation(
                operation_type="write",
                source_path=str(path),
                success=True,
                message=f"写入成功，{len(content)} 字符",
            ))
            
            return True
            
        except Exception as e:
            print(f"写入失败: {e}")
            return False
    
    def list_directory(self, dir_path: str = ".", 
                      pattern: str = "*") -> List[Dict]:
        """列出目录内容"""
        path = Path(dir_path)
        
        if not self._is_allowed_path(path):
            return [{"error": "无权访问该路径"}]
        
        results = []
        
        for item in path.glob(pattern):
            stat = item.stat()
            results.append({
                "name": item.name,
                "type": "directory" if item.is_dir() else "file",
                "size": stat.st_size,
                "modified": datetime.fromtimestamp(stat.st_mtime).isoformat(),
                "path": str(item),
            })
        
        return sorted(results, key=lambda x: x["name"])
    
    def copy_file(self, source: str, destination: str) -> bool:
        """复制文件"""
        src = Path(source)
        dst = Path(destination)
        
        if not self._is_allowed_path(src) or not self._is_allowed_path(dst):
            print("错误: 无权访问该路径")
            return False
        
        try:
            shutil.copy2(src, dst)
            
            self.operations.append(FileOperation(
                operation_type="copy",
                source_path=str(src),
                destination_path=str(dst),
                success=True,
            ))
            
            return True
            
        except Exception as e:
            print(f"复制失败: {e}")
            return False
    
    def move_file(self, source: str, destination: str) -> bool:
        """移动文件"""
        src = Path(source)
        dst = Path(destination)
        
        if not self._is_allowed_path(src) or not self._is_allowed_path(dst):
            print("错误: 无权访问该路径")
            return False
        
        try:
            shutil.move(src, dst)
            
            self.operations.append(FileOperation(
                operation_type="move",
                source_path=str(src),
                destination_path=str(dst),
                success=True,
            ))
            
            return True
            
        except Exception as e:
            print(f"移动失败: {e}")
            return False
    
    def delete_file(self, file_path: str) -> bool:
        """删除文件"""
        path = Path(file_path)
        
        if not self._is_allowed_path(path):
            print("错误: 无权访问该路径")
            return False
        
        try:
            if path.is_file():
                path.unlink()
            elif path.is_dir():
                shutil.rmtree(path)
            
            self.operations.append(FileOperation(
                operation_type="delete",
                source_path=str(path),
                success=True,
            ))
            
            return True
            
        except Exception as e:
            print(f"删除失败: {e}")
            return False
    
    def search_files(self, directory: str, keyword: str,
                    search_content: bool = False) -> List[Dict]:
        """搜索文件"""
        dir_path = Path(directory)
        
        if not self._is_allowed_path(dir_path):
            return [{"error": "无权访问该路径"}]
        
        results = []
        
        for path in dir_path.rglob("*"):
            if path.is_file():
                # 按文件名搜索
                if keyword.lower() in path.name.lower():
                    results.append({
                        "path": str(path),
                        "match_type": "filename",
                        "name": path.name,
                    })
                
                # 按内容搜索
                elif search_content:
                    try:
                        content = path.read_text(encoding='utf-8', errors='ignore')
                        if keyword.lower() in content.lower():
                            results.append({
                                "path": str(path),
                                "match_type": "content",
                                "name": path.name,
                            })
                    except:
                        pass
        
        return results
    
    def get_file_info(self, file_path: str) -> Dict:
        """获取文件信息"""
        path = Path(file_path)
        
        if not path.exists():
            return {"error": "文件不存在"}
        
        stat = path.stat()
        
        info = {
            "name": path.name,
            "path": str(path),
            "type": "directory" if path.is_dir() else "file",
            "size": stat.st_size,
            "size_human": self._format_size(stat.st_size),
            "created": datetime.fromtimestamp(stat.st_ctime).isoformat(),
            "modified": datetime.fromtimestamp(stat.st_mtime).isoformat(),
            "accessed": datetime.fromtimestamp(stat.st_atime).isoformat(),
        }
        
        if path.is_file():
            info["extension"] = path.suffix
            info["md5"] = self._calculate_md5(path)
        
        return info
    
    def _format_size(self, size: int) -> str:
        """格式化文件大小"""
        for unit in ['B', 'KB', 'MB', 'GB']:
            if size < 1024:
                return f"{size:.2f} {unit}"
            size /= 1024
        return f"{size:.2f} TB"
    
    def _calculate_md5(self, file_path: Path) -> str:
        """计算 MD5"""
        hash_md5 = hashlib.md5()
        try:
            with open(file_path, "rb") as f:
                for chunk in iter(lambda: f.read(4096), b""):
                    hash_md5.update(chunk)
            return hash_md5.hexdigest()
        except:
            return ""
    
    def analyze_with_llm(self, file_path: str) -> str:
        """使用 LLM 分析文件"""
        content = self.read_file(file_path)
        
        if content.startswith("错误"):
            return content
        
        prompt = f"""分析以下文件内容。

文件路径：{file_path}
内容：
{content[:5000]}

请提供：
1. 文件类型和用途
2. 内容摘要
3. 关键信息
4. 建议的处理方式"""
        
        response = self.client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            messages=[{"role": "user", "content": prompt}],
        )
        
        return response.content[0].text


def demo_file_agent():
    """演示文件操作 Agent"""
    print("=" * 60)
    print("文件操作 Agent 演示")
    print("=" * 60)
    
    # 创建临时目录
    test_dir = Path("file_agent_test")
    test_dir.mkdir(exist_ok=True)
    
    agent = FileSystemAgent(str(test_dir))
    
    # 创建示例文件
    print("\n创建示例文件...")
    agent.write_file("test1.txt", "这是第一个测试文件。\n包含一些示例内容。")
    agent.write_file("test2.txt", "这是第二个测试文件。\n不同的内容。")
    agent.write_file("data.json", json.dumps({"name": "test", "value": 42}, indent=2))
    
    # 列出目录
    print("\n目录内容:")
    items = agent.list_directory(".")
    for item in items:
        print(f"  {item['type']}: {item['name']} ({item['size']} bytes)")
    
    # 搜索文件
    print("\n搜索包含 '测试' 的文件:")
    results = agent.search_files(".", "测试")
    for r in results:
        print(f"  {r['name']} ({r['match_type']})")
    
    # 获取文件信息
    print("\n文件信息:")
    info = agent.get_file_info("test1.txt")
    print(f"  名称: {info['name']}")
    print(f"  大小: {info['size_human']}")
    print(f"  修改时间: {info['modified']}")
    
    # 分析文件
    print("\nLLM 分析:")
    analysis = agent.analyze_with_llm("test1.txt")
    print(analysis[:300])
    
    # 清理
    import shutil
    shutil.rmtree(test_dir)


if __name__ == "__main__":
    demo_file_agent()
```

### 示例 2：批量文件处理器

```python
"""
批量文件处理器
高效处理大量文件
"""

import os
import json
import csv
from pathlib import Path
from typing import Dict, List, Callable, Any
from dataclasses import dataclass, field
from concurrent.futures import ThreadPoolExecutor
import anthropic


@dataclass
class ProcessingResult:
    """处理结果"""
    file_path: str
    success: bool
    result: Any = None
    error: str = ""
    duration_ms: float = 0


class BatchFileProcessor:
    """批量文件处理器"""
    
    def __init__(self, max_workers: int = 4):
        self.max_workers = max_workers
        self.client = anthropic.Anthropic(
            api_key=os.environ.get("ANTHROPIC_API_KEY")
        )
    
    def process_batch(self, files: List[str], 
                     processor: Callable[[str], Any]) -> List[ProcessingResult]:
        """批量处理文件"""
        results = []
        
        with ThreadPoolExecutor(max_workers=self.max_workers) as executor:
            futures = {
                executor.submit(self._process_single, f, processor): f
                for f in files
            }
            
            for future in futures:
                file_path = futures[future]
                try:
                    result = future.result()
                    results.append(result)
                except Exception as e:
                    results.append(ProcessingResult(
                        file_path=file_path,
                        success=False,
                        error=str(e),
                    ))
        
        return results
    
    def _process_single(self, file_path: str,
                       processor: Callable[[str], Any]) -> ProcessingResult:
        """处理单个文件"""
        import time
        start_time = time.time()
        
        try:
            result = processor(file_path)
            duration = (time.time() - start_time) * 1000
            
            return ProcessingResult(
                file_path=file_path,
                success=True,
                result=result,
                duration_ms=duration,
            )
        except Exception as e:
            duration = (time.time() - start_time) * 1000
            return ProcessingResult(
                file_path=file_path,
                success=False,
                error=str(e),
                duration_ms=duration,
            )
    
    def count_lines(self, file_path: str) -> int:
        """统计行数"""
        with open(file_path, 'r', encoding='utf-8', errors='ignore') as f:
            return sum(1 for _ in f)
    
    def extract_text(self, file_path: str) -> str:
        """提取文本"""
        with open(file_path, 'r', encoding='utf-8', errors='ignore') as f:
            return f.read()
    
    def summarize_batch(self, results: List[ProcessingResult]) -> Dict:
        """汇总批量处理结果"""
        successful = [r for r in results if r.success]
        failed = [r for r in results if not r.success]
        
        return {
            "total_files": len(results),
            "successful": len(successful),
            "failed": len(failed),
            "total_duration_ms": sum(r.duration_ms for r in results),
            "avg_duration_ms": (
                sum(r.duration_ms for r in results) / len(results)
                if results else 0
            ),
            "failed_files": [r.file_path for r in failed],
        }


def demo_batch_processor():
    """演示批量处理器"""
    print("=" * 60)
    print("批量文件处理器演示")
    print("=" * 60)
    
    # 创建测试目录
    test_dir = Path("batch_test")
    test_dir.mkdir(exist_ok=True)
    
    # 创建测试文件
    for i in range(10):
        (test_dir / f"file_{i}.txt").write_text(
            f"这是文件 {i} 的内容。\n" * (i + 1),
            encoding='utf-8'
        )
    
    processor = BatchFileProcessor(max_workers=4)
    
    # 获取文件列表
    files = [str(f) for f in test_dir.glob("*.txt")]
    print(f"\n处理 {len(files)} 个文件...")
    
    # 批量统计行数
    results = processor.process_batch(files, processor.count_lines)
    
    # 显示结果
    print("\n处理结果:")
    for r in results:
        status = "✓" if r.success else "✗"
        print(f"  {status} {Path(r.file_path).name}: {r.result if r.success else r.error}")
    
    # 汇总
    summary = processor.summarize_batch(results)
    print(f"\n汇总:")
    print(f"  成功: {summary['successful']}/{summary['total_files']}")
    print(f"  总耗时: {summary['total_duration_ms']:.2f}ms")
    
    # 清理
    import shutil
    shutil.rmtree(test_dir)


if __name__ == "__main__":
    demo_batch_processor()
```

### 示例 3：智能文件整理 Agent

前面两个示例分别展示了基础文件操作和批量处理。但真实的文件管理场景中，用户的需求往往是"帮我整理一下这个目录"——这是一个模糊的指令，需要 Agent 自己判断如何分类、如何命名。下面这个示例展示了一个结合 LLM 的智能文件整理 Agent，它能够理解文件内容，自动进行分类和整理。

```python
"""
智能文件整理 Agent
结合 LLM 理解文件内容，自动分类和整理
"""

import os
import json
import shutil
from pathlib import Path
from typing import Dict, List, Optional
from dataclasses import dataclass, field
from datetime import datetime
import anthropic


@dataclass
class FileClassification:
    """文件分类结果"""
    file_path: str
    category: str
    suggested_name: str
    summary: str
    confidence: float


class SmartFileOrganizer:
    """智能文件整理 Agent。

    工作流程：
    1. 扫描目标目录，列出所有文件
    2. 对每个文件，使用 LLM 分析内容并分类
    3. 根据分类结果创建子目录
    4. 移动文件到对应目录
    5. 生成整理报告
    """

    # 预定义的分类规则
    DEFAULT_CATEGORIES = {
        "文档": [".txt", ".md", ".doc", ".docx", ".pdf"],
        "图片": [".jpg", ".jpeg", ".png", ".gif", ".bmp", ".svg"],
        "代码": [".py", ".js", ".ts", ".java", ".cpp", ".go"],
        "数据": [".csv", ".json", ".xml", ".yaml", ".sql"],
        "压缩包": [".zip", ".tar", ".gz", ".rar", ".7z"],
        "音频": [".mp3", ".wav", ".flac", ".aac"],
        "视频": [".mp4", ".avi", ".mkv", ".mov"],
    }

    def __init__(self, base_dir: str, api_key: Optional[str] = None):
        self.base_dir = Path(base_dir).resolve()
        self.client = anthropic.Anthropic(
            api_key=api_key or os.environ.get("ANTHROPIC_API_KEY")
        )
        self.classifications: List[FileClassification] = []
        self.operations_log: List[Dict] = []

    def scan_directory(self) -> List[Path]:
        """扫描目录，返回所有文件。"""
        files = []
        for item in self.base_dir.rglob("*"):
            if item.is_file() and not item.name.startswith("."):
                files.append(item)
        return files

    def classify_by_extension(self, file_path: Path) -> str:
        """根据文件扩展名分类。"""
        suffix = file_path.suffix.lower()
        for category, extensions in self.DEFAULT_CATEGORIES.items():
            if suffix in extensions:
                return category
        return "其他"

    def classify_with_llm(self, file_path: Path) -> FileClassification:
        """使用 LLM 分析文件内容并分类。

        适用于文本类文件，可以深入理解内容语义。
        """
        try:
            content = file_path.read_text(encoding="utf-8", errors="ignore")
            # 只发送前 2000 字符给 LLM
            content_preview = content[:2000]
        except Exception:
            content_preview = "(无法读取文件内容)"

        prompt = f"""分析以下文件并给出分类建议。

文件名：{file_path.name}
文件扩展名：{file_path.suffix}
文件内容预览：
{content_preview}

请返回 JSON 格式：
{{
    "category": "文件类别（如：工作文档、学习笔记、个人随笔、代码、数据、配置文件等）",
    "suggested_name": "建议的规范文件名（如：2026-05-项目总结报告.md）",
    "summary": "一句话摘要",
    "confidence": 0.0-1.0
}}"""

        try:
            response = self.client.messages.create(
                model="claude-sonnet-4-20250514",
                max_tokens=512,
                messages=[{"role": "user", "content": prompt}],
            )

            text = response.content[0].text
            start = text.find("{")
            end = text.rfind("}") + 1
            if start >= 0 and end > start:
                data = json.loads(text[start:end])
                return FileClassification(
                    file_path=str(file_path),
                    category=data.get("category", "未分类"),
                    suggested_name=data.get("suggested_name", file_path.name),
                    summary=data.get("summary", ""),
                    confidence=data.get("confidence", 0.5),
                )
        except Exception as e:
            print(f"  LLM 分析失败: {e}")

        # 回退到扩展名分类
        category = self.classify_by_extension(file_path)
        return FileClassification(
            file_path=str(file_path),
            category=category,
            suggested_name=file_path.name,
            summary="",
            confidence=0.3,
        )

    def organize_files(self, use_llm: bool = True,
                       min_confidence: float = 0.5) -> Dict:
        """整理目录中的文件。

        Args:
            use_llm: 是否使用 LLM 进行智能分类
            min_confidence: 最低置信度阈值

        Returns:
            整理报告
        """
        files = self.scan_directory()
        print(f"扫描到 {len(files)} 个文件")

        # 分类每个文件
        for file_path in files:
            print(f"\n分析: {file_path.name}")

            if use_llm:
                classification = self.classify_with_llm(file_path)
            else:
                category = self.classify_by_extension(file_path)
                classification = FileClassification(
                    file_path=str(file_path),
                    category=category,
                    suggested_name=file_path.name,
                    summary="",
                    confidence=0.3,
                )

            self.classifications.append(classification)
            print(f"  分类: {classification.category} (置信度: {classification.confidence:.1%})")

        # 创建目录并移动文件
        moved_count = 0
        for cls in self.classifications:
            if cls.confidence < min_confidence:
                continue

            # 创建分类目录
            category_dir = self.base_dir / cls.category
            category_dir.mkdir(exist_ok=True)

            src = Path(cls.file_path)
            dst = category_dir / cls.suggested_name

            # 避免覆盖同名文件
            if dst.exists() and dst != src:
                stem = dst.stem
                suffix = dst.suffix
                counter = 1
                while dst.exists():
                    dst = category_dir / f"{stem}_{counter}{suffix}"
                    counter += 1

            try:
                shutil.move(str(src), str(dst))
                moved_count += 1
                self.operations_log.append({
                    "action": "move",
                    "from": str(src),
                    "to": str(dst),
                    "category": cls.category,
                })
                print(f"  已移动: {src.name} -> {cls.category}/{dst.name}")
            except Exception as e:
                print(f"  移动失败: {e}")

        # 生成报告
        report = {
            "total_files": len(files),
            "moved_files": moved_count,
            "categories": {},
        }
        for cls in self.classifications:
            cat = cls.category
            if cat not in report["categories"]:
                report["categories"][cat] = 0
            report["categories"][cat] += 1

        return report


def demo_smart_organizer():
    """演示智能文件整理 Agent。"""
    print("=" * 60)
    print("智能文件整理 Agent 演示")
    print("=" * 60)

    # 创建测试目录
    test_dir = Path("organizer_test")
    test_dir.mkdir(exist_ok=True)

    # 创建各种类型的测试文件
    (test_dir / "会议记录_20260501.txt").write_text(
        "会议主题：Q2 季度目标讨论\n参会人员：张三、李四\n结论：......",
        encoding="utf-8",
    )
    (test_dir / "学习笔记_Python异步编程.md").write_text(
        "# Python 异步编程\n\n## asyncio 基础\n\nasync/await 是 Python 的异步编程语法......",
        encoding="utf-8",
    )
    (test_dir / "data_export.csv").write_text(
        "姓名,部门,工资\n张三,技术部,15000\n李四,产品部,13000",
        encoding="utf-8",
    )
    (test_dir / "photo_20260501.jpg").write_bytes(b"\xff\xd8\xff\xe0" + b"\x00" * 100)

    # 运行整理
    organizer = SmartFileOrganizer(str(test_dir))
    report = organizer.organize_files(use_llm=False)  # 演示时不使用 LLM

    print(f"\n整理报告:")
    print(f"  总文件数: {report['total_files']}")
    print(f"  已移动: {report['moved_files']}")
    for cat, count in report['categories'].items():
        print(f"  {cat}: {count} 个文件")

    # 清理
    shutil.rmtree(test_dir)


if __name__ == "__main__":
    demo_smart_organizer()
```

这个示例展示了文件操作 Agent 的一个完整工作流程：扫描、分析、分类、移动、报告。它结合了基于规则的分类（扩展名匹配）和基于 LLM 的智能分类（内容理解），在 LLM 不可用时也能回退到基础分类。在实际应用中，这种分层策略非常实用——规则处理简单的情况，LLM 处理需要语义理解的复杂情况。

---

## 案例分析

### 案例 1：个人文件整理助手

**场景**：帮助用户整理混乱的文件系统。

想象这样一个场景：你的电脑"下载"目录里堆积了上千个文件——有从网上下载的图片、有工作中保存的文档、有临时存放的数据文件、有很久以前下载却忘记删除的安装包。你想要整理这些文件，但面对这么多文件，光是看一遍就需要几个小时。

**Agent 功能**：
1. 扫描指定目录（递归扫描所有文件，记录文件名、大小、修改时间、扩展名等信息）
2. 识别文件类型（不仅看扩展名，还通过读取文件头来确认真实类型，防止扩展名被篡改）
3. 自动分类移动（根据预设规则和 LLM 分析结果，将文件移动到对应的子目录）
4. 重命名规范化（根据文件内容和时间信息，生成规范的文件名）
5. 检测重复文件（通过计算文件的 MD5 哈希值来识别内容完全相同的文件）

**实现要点**：文件整理 Agent 的核心挑战是如何准确地分类文件。纯基于扩展名的分类太粗糙，纯基于 LLM 的分类太慢（每个文件都要调用 API）。最佳实践是分层处理：先用扩展名做粗分类，再对"其他"类别的文件用 LLM 做细分类。

### 案例 2：文档处理系统

**场景**：批量处理文档文件。

**Agent 功能**：
1. 提取文档内容（支持 PDF、Word、Excel、Markdown 等多种格式）
2. 生成文档摘要（使用 LLM 对长文档生成结构化摘要）
3. 转换文档格式（如 Markdown 转 HTML、CSV 转 JSON）
4. 提取关键信息（如从合同中提取甲乙方信息、金额、日期等）
5. 分类归档（根据文档内容自动分类并归档）

**实现要点**：文档处理的挑战在于格式多样性。每种格式需要不同的解析库——PDF 用 PyPDF2 或 pdfplumber，Word 用 python-docx，Excel 用 openpyxl。Agent 需要根据文件扩展名自动选择合适的解析器，并在解析失败时优雅地降级处理。

---

## 常见坑

### 坑 1：文件编码问题

不同文件可能使用不同的编码。

**解决方案**：自动检测编码，提供编码转换。

### 坑 2：大文件处理

处理大文件可能消耗大量内存。

**解决方案**：使用流式处理，分块读取。

### 坑 3：并发安全

多个进程同时操作文件可能导致冲突。

**解决方案**：使用文件锁，实现乐观锁。

### 坑 4：路径处理

不同操作系统的路径格式不同。

**解决方案**：使用 pathlib，统一路径处理。

### 坑 5：错误恢复

文件操作失败后需要恢复。

**解决方案**：实现事务机制，支持回滚。

---

## 练习题

### 练习 1：基础文件操作（难度：初级）
实现基础的文件操作：
- 读取文件（处理不同编码的文件）
- 写入文件（自动创建不存在的目录）
- 列出目录（支持按名称、大小、时间排序）

提示：使用 Python 的 pathlib 模块处理路径，它比 os.path 更现代、更安全。对于编码检测，可以使用 chardet 库。

### 练习 2：文件搜索（难度：初级）
实现文件搜索功能：
- 按名称搜索（支持通配符和正则表达式）
- 按内容搜索（搜索文件内部的文本）
- 按类型搜索（按文件扩展名过滤）
- 按时间范围搜索（查找最近 N 天修改过的文件）

提示：使用 pathlib 的 `glob()` 和 `rglob()` 方法进行文件名搜索，使用逐文件读取进行内容搜索。对于大文件，建议使用流式读取避免内存问题。

### 练习 3：批量处理（难度：中级）
实现批量文件处理：
- 使用 `concurrent.futures.ThreadPoolExecutor` 并行处理
- 进度追踪（使用 tqdm 库显示进度条）
- 结果汇总（统计成功/失败数量和耗时）

提示：在批量处理时，要注意异常处理——单个文件的处理失败不应该影响其他文件。建议在每个文件的处理外层包裹 try-except。

### 练习 4：文件格式转换（难度：中级）
实现文件格式转换：
- CSV 转 JSON（处理中文编码和嵌套数据）
- JSON 转 CSV（处理嵌套 JSON 的展平）
- Markdown 转 HTML（使用 markdown 库）
- 批量格式转换（一次转换整个目录的文件）

提示：CSV 转 JSON 时，使用 `csv.DictReader` 可以自动将每行映射为字典。JSON 转 CSV 时，如果数据有嵌套，需要先展平（flatten）为一级字典。

### 练习 5：文件分析（难度：中级）
实现文件分析功能：
- 统计信息（文件数量、总大小、类型分布）
- 内容分析（词频统计、关键词提取）
- 重复检测（基于 MD5 哈希的文件去重）

提示：重复检测的关键是高效计算大文件的哈希值——使用分块读取（每次读 4KB）来避免将整个文件加载到内存。

### 练习 6：完整的文件管理 Agent（难度：高级）
构建一个完整的文件管理 Agent，集成 LLM 能力。

要求：结合示例 3 的 SmartFileOrganizer，实现一个能完成以下任务的 Agent：
1. 接受用户的自然语言指令（如"帮我整理下载目录，按文件类型分类"）
2. 使用 LLM 理解用户意图并规划操作步骤
3. 安全地执行文件操作（有权限检查和操作审计）
4. 支持撤销操作（将文件移动到回收站而非直接删除）
5. 生成操作报告（记录所有操作，便于回溯）

提示：实现撤销功能的一个简单方法是维护一个操作日志，在日志中记录每个操作的原始位置和目标位置，这样就可以根据日志来还原操作。

---

## 实战任务

### 任务 1：构建个人文件整理助手（中等难度）

**功能需求：**
1. 扫描目录
2. 自动分类
3. 重命名
4. 去重

### 任务 2：构建文档处理系统（高难度）

**功能需求：**
1. 多格式支持
2. 内容提取
3. 批量处理
4. 生成报告

---

## 本章小结

本章我们学习了 Agent 与文件系统的结合。我们从以下几个方面进行了探索：

**核心能力**：理解了文件读写、内容理解、文件组织、批量处理和搜索检索等核心能力。

**安全考虑**：掌握了权限控制、操作审计、备份机制等安全措施。

**格式支持**：了解了文本文件、文档文件、媒体文件等多种格式的处理方法。

**智能管理**：学会了自动分类、重复检测、版本管理等智能管理功能。

文件操作 Agent 是提升个人和企业效率的重要工具。通过自动化日常的文件操作，可以节省大量时间和精力。

在下一章中，我们将进行 Agent 框架对比与选型指南，帮助你选择最适合的 Agent 框架。

---

## 延伸阅读

1. Python pathlib 官方文档
2. 文件系统最佳实践
3. 批量处理设计模式
4. 文件安全操作指南
5. 文件格式处理库
