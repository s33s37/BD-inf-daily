"""
孟加拉商业情报日报系统 - 主程序
每日自动采集、分析、生成报告并推送
"""
import os
import sys
import json
from datetime import datetime

from src.crawler import RSSCrawler
from src.analyzer import IntelligenceAnalyzer
from src.generator import ReportGenerator
from src.wechat import WeChatNotifier


def main():
    print("=" * 60)
    print("  孟加拉商业情报日报系统")
    print("  Bangladesh Business Intelligence Daily")
    print("=" * 60)
    print(f"  运行时间: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
    print()

    # Step 1: 采集
    print("[Step 1/4] 正在采集RSS源...")
    crawler = RSSCrawler()
    crawl_result = crawler.crawl(target_count=50)
    items = crawl_result["items"]

    if len(items) < 30:
        print(f"⚠️ 警告: 仅采集到 {len(items)} 条新闻，低于目标30条")

    # 保存原始数据
    os.makedirs("data", exist_ok=True)
    with open(f"data/raw_{datetime.now().strftime('%Y%m%d')}.json", "w", encoding="utf-8") as f:
        json.dump(crawl_result, f, ensure_ascii=False, indent=2)

    # Step 2: AI分析
    print("
[Step 2/4] 正在进行AI分析...")
    try:
        analyzer = IntelligenceAnalyzer()
        analyses = analyzer.analyze_batch(items, batch_size=5)

        # 保存分析结果
        with open(f"data/analyzed_{datetime.now().strftime('%Y%m%d')}.json", "w", encoding="utf-8") as f:
            json.dump(analyses, f, ensure_ascii=False, indent=2)
    except Exception as e:
        print(f"❌ AI分析失败: {e}")
        print("   请检查 OPENAI_API_KEY 或 DEEPSEEK_API_KEY 环境变量")
        sys.exit(1)

    # Step 3: 生成统计
    print("
[Step 3/4] 正在生成统计...")
    stats = analyzer.get_stats(analyses)
    stats["model"] = analyzer.model
    print(f"  产业分类统计: {stats['by_industry']}")
    print(f"  影响分布: {stats['by_impact']}")
    print(f"  平均置信度: {stats['avg_confidence']:.0%}")

    # Step 4: 生成报告
    print("
[Step 4/4] 正在生成HTML报告...")
    generator = ReportGenerator()
    report_path = generator.generate_html(analyses, stats, output_dir="docs")

    # Step 5: 微信推送
    print("
[Step 5/5] 正在推送微信通知...")
    notifier = WeChatNotifier()

    # GitHub Pages URL
    github_repo = os.environ.get("GITHUB_REPOSITORY", "")
    if github_repo:
        report_url = f"https://{github_repo.split('/')[0]}.github.io/{github_repo.split('/')[1]}/"
    else:
        report_url = "https://your-username.github.io/bangladesh-intelligence-daily/"

    notifier.send_daily_summary(stats, analyses[:10], report_url)

    # 最终统计
    print("
" + "=" * 60)
    print("  日报生成完成!")
    print(f"  采集新闻: {len(items)} 条")
    print(f"  成功分析: {len(analyses)} 条")
    print(f"  风险预警: {stats['risk_count']} 条")
    print(f"  政策动态: {stats['policy_count']} 条")
    print(f"  涉华情报: {stats['china_related']} 条")
    print(f"  报告路径: {report_path}")
    print(f"  访问地址: {report_url}")
    print("=" * 60)


if __name__ == "__main__":
    main()
