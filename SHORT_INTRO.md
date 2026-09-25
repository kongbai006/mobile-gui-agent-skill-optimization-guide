# 移动端GUI Agent技能优化训练方案

分享一套完整的移动端GUI Agent技能优化训练方法论，从纯VLM截图识图（单步5-8秒、高Token、70%准确率）迭代到四级降级识别架构（指纹缓存→控件树→模板匹配→VLM兜底），实现单步识别最快10ms、Token消耗下降95%、任务总耗时减少58%、识别准确率提升至97%。

项目完整记录了v1.0到v3.0的全部优化历程：控件树替代截图、事件驱动等待替代固定sleep、局部指纹缓存解决动态页面问题、ImageMagick模板匹配接管无控件场景。包含完整的技术方案、量化对比数据、工具脚本和未来演进方向，适合做安卓自动化、端侧Agent、手机助手的开发者参考。

详细内容见GitHub仓库：https://github.com/kongbai006/mobile-gui-agent-skill-optimization-guide
