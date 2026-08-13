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
        # 境界列表（顺序固定）
        self.realms = [
            "武者", "化气", "锻心", "金丹", "元婴",
            "凝神", "真灵", "渡劫", "天仙", "仙帝"
        ]
        # 每个境界的最大层数（索引对应）
        self.max_levels = [9, 9, 9, 9, 9, 9, 9, 9, 20, 9]
        # 每个境界的基础修为和每层增量（经验值需求）
        # 需求 = base + (level-1) * add
        self.base_exp = [10, 100, 200, 400, 800, 1600, 3200, 6400, 12800, 25600]
        self.add_exp  = [10, 20,  40,  80, 160,  320,  640, 1280,  2560,  5120]

        self.user_data = {}  # 内存存储

    def _get_user(self, user_id: str):
        if user_id not in self.user_data:
            self.user_data[user_id] = {
                "realm_idx": 0,      # 武者
                "level": 1,          # 1层
                "exp": 0,
                "total_exp": 0
            }
        return self.user_data[user_id]

    def _get_realm_name(self, idx):
        return self.realms[idx]

    def _get_max_level(self, idx):
        return self.max_levels[idx]

    def _get_exp_required(self, idx, level):
        """计算当前层升级所需修为"""
        return self.base_exp[idx] + (level - 1) * self.add_exp[idx]

    @filter.command("修炼")
    async def cmd_cultivate(self, event: AstrMessageEvent):
        """指令：/修炼 — 增长修为（渡劫境不可用）"""
        user = self._get_user(event.get_sender_id())
        idx = user["realm_idx"]
        realm = self._get_realm_name(idx)

        if realm == "渡劫":
            yield event.plain_result("⚡ 你身处渡劫境，无法静心修炼，必须马上渡劫！")
            return

        gain = random.randint(10, 30)
        user["exp"] += gain
        user["total_exp"] += gain
        logger.info(f"{event.get_sender_name()} 修炼 +{gain} 修为")
        yield event.plain_result(f"🧘 你静心修炼，获得 {gain} 点修为！当前修为：{user['exp']}")

    @filter.command("突破")
    async def cmd_breakthrough(self, event: AstrMessageEvent):
        """指令：/突破 — 突破层数或境界（含渡劫特殊处理）"""
        user = self._get_user(event.get_sender_id())
        idx = user["realm_idx"]
        level = user["level"]
        realm = self._get_realm_name(idx)
        max_lv = self._get_max_level(idx)

        # ----- 渡劫境特殊逻辑 -----
        if realm == "渡劫":
            if level >= max_lv:  # 9层渡劫圆满，直接晋升天仙
                user["realm_idx"] = self.realms.index("天仙")
                user["level"] = 1
                user["exp"] = 0
                yield event.plain_result("🌌 九重雷劫尽数渡过！你飞升为 **天仙**！")
                return

            # 尝试渡劫（每层一劫）
            success_rate = 0.6 + (level - 1) * 0.05  # 越高层成功率略增
            if random.random() < success_rate:
                user["level"] += 1
                yield event.plain_result(f"⚡ 成功渡过第 {level} 重雷劫！当前渡劫层数：{user['level']}/9")
            else:
                # 失败退回真灵（真灵9层）
                user["realm_idx"] = self.realms.index("真灵")
                user["level"] = 9
                user["exp"] = 0
                yield event.plain_result("💥 渡劫失败！肉身崩毁，神魂退回 **真灵九层**！")
            return

        # ----- 普通境界突破 -----
        required = self._get_exp_required(idx, level)
        if user["exp"] < required:
            yield event.plain_result(
                f"⚠️ 修为不足，突破 {realm}{level}层 需要 {required} 修为，当前 {user['exp']}"
            )
            return

        # 修为足够
        if level < max_lv:
            # 升一层
            user["level"] += 1
            user["exp"] -= required
            yield event.plain_result(
                f"🎉 突破成功！当前境界：{realm}{user['level']}层，剩余修为：{user['exp']}"
            )
        else:
            # 当前境界已满，突破至下一个大境界
            if idx >= len(self.realms) - 1:
                yield event.plain_result("🏁 你已抵达仙帝巅峰，世间再无更高境界！")
                return
            next_idx = idx + 1
            next_realm = self._get_realm_name(next_idx)
            user["realm_idx"] = next_idx
            user["level"] = 1
            user["exp"] -= required
            yield event.plain_result(
                f"🌟 大境界突破！你踏入 **{next_realm}** 一层，剩余修为：{user['exp']}"
            )

    @filter.command("修为")
    async def cmd_status(self, event: AstrMessageEvent):
        """指令：/修为 — 查看当前状态"""
        user = self._get_user(event.get_sender_id())
        idx = user["realm_idx"]
        level = user["level"]
        realm = self._get_realm_name(idx)
        required = self._get_exp_required(idx, level)
        next_realm = self._get_realm_name(idx+1) if idx < len(self.realms)-1 else "已至巅峰"
        yield event.plain_result(
            f"📊 当前境界：**{realm} {level}层**\n"
            f"修为：{user['exp']} / {required}\n"
            f"下一层所需修为：{required}\n"
            f"下一大境界：{next_realm}\n"
            f"总修行：{user['total_exp']}"
        )
