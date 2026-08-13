# astrbot_plugin_asd
喵喵喵
# main.py
import random
from astrbot.api.event import filter, AstrMessageEvent
from astrbot.api.star import Context, Star
from astrbot.api import logger

class XiuxianPlugin(Star):
    def __init__(self, context: Context):
        super().__init__(context)
        # 内存存储用户数据（重启会丢失，生产环境可替换为数据库）
        self.user_data = {}  
        # 境界体系
        self.realm_map = {
            "凡人": {"exp_needed": 100, "next": "炼气"},
            "炼气": {"exp_needed": 300, "next": "筑基"},
            "筑基": {"exp_needed": 800, "next": "金丹"},
            "金丹": {"exp_needed": 2000, "next": "元婴"},
            "元婴": {"exp_needed": 5000, "next": "化神"},
            "化神": {"exp_needed": 99999, "next": "大乘"}  # 封顶
        }

    def _get_user(self, user_id: str):
        if user_id not in self.user_data:
            self.user_data[user_id] = {
                "realm": "凡人",
                "exp": 0,
                "total_exp": 0
            }
        return self.user_data[user_id]

    @filter.command("修炼")
    async def cmd_cultivate(self, event: AstrMessageEvent):
        """指令：/修炼 — 随机增长 10~30 修为"""
        user = self._get_user(event.get_sender_id())
        gain = random.randint(10, 30)
        user["exp"] += gain
        user["total_exp"] += gain
        logger.info(f"{event.get_sender_name()} 修炼 +{gain} 修为")
        yield event.plain_result(f"🧘 你静心修炼，获得 {gain} 点修为！当前修为：{user['exp']}")

    @filter.command("突破")
    async def cmd_breakthrough(self, event: AstrMessageEvent):
        """指令：/突破 — 尝试突破至下一境界"""
        user = self._get_user(event.get_sender_id())
        current_realm = user["realm"]
        if current_realm == "化神":
            yield event.plain_result(" 你已抵达化神巅峰，世间再无敌手！")
            return

        needed = self.realm_map[current_realm]["exp_needed"]
        if user["exp"] < needed:
            yield event.plain_result(f" 修为不足，突破 {current_realm}→{self.realm_map[current_realm]['next']} 需要 {needed} 修为，当前仅 {user['exp']}")
            return

        # 突破成功（添加一点随机失败感，更修仙）
        if random.random() < 0.1:  # 10% 失败率
            loss = random.randint(5, 20)
            user["exp"] = max(0, user["exp"] - loss)
            yield event.plain_result(f" 突破失败！走火入魔，损失 {loss} 修为！当前修为：{user['exp']}")
            return

        # 成功
        new_realm = self.realm_map[current_realm]["next"]
        user["realm"] = new_realm
        user["exp"] -= needed  # 扣除所需修为
        yield event.plain_result(f" 恭喜突破至 **{new_realm}**！剩余修为：{user['exp']}")

    @filter.command("修为")
    async def cmd_status(self, event: AstrMessageEvent):
        """指令：/修为 — 查看当前境界和修为"""
        user = self._get_user(event.get_sender_id())
        current = user["realm"]
        next_realm = self.realm_map[current]["next"] if current != "化神" else "已臻化神"
        needed = self.realm_map[current]["exp_needed"] if current != "化神" else "无"
        yield event.plain_result(
            f" 境界：**{current}**\n"
            f"修为：{user['exp']}\n"
            f"下一境界：{next_realm}（需求 {needed} 修为）\n"
            f"总修行：{user['total_exp']}"
        )
