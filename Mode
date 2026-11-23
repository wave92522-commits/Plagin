import time
import random
from collections import deque

from base_plugin import BasePlugin, HookResult, HookStrategy
from ui.settings import Header, Switch, Input, Divider
from ui.bulletin import BulletinHelper
from android_utils import run_on_ui_thread, log
from client_utils import get_user_config, get_messages_controller

from java.util import ArrayList
from org.telegram.tgnet import TLRPC
from org.telegram.tgnet.tl import TL_stars
from org.telegram.messenger import MessageObject
from org.telegram.ui.Stars import StarsController

__id__ = "fake_gifts_visual"
__name__ = "Fake Gifts & Stars"
__description__ = "Визуальные фейк-подарки, фейк-баланс звёзд и фейк-список подарков. Только локально, для развлечения."
__author__ = "you"
__version__ = "1.0.0"
__icon__ = "exteraPlugins/1"
__min_version__ = "11.12.0"


class FakeGiftsPlugin(BasePlugin):

    def on_plugin_load(self):
        self._pending = deque()
        self.add_on_send_message_hook()
        self.add_hook("getStarsStatus", match_substring=True)
        self.add_hook("getSavedStarGifts", match_substring=True)
        self.add_hook("TL_users_getFullUser")
        self.add_hook("TL_updateStarsBalance")

    # ---------- настройки ----------

    def create_settings(self):
        return [
            Header(text="Фейк звёзды"),
            Switch(key="fake_stars", text="Подменять баланс звёзд", default=True),
            Input(key="stars_amount", text="Количество звёзд", default="99999"),
            Divider(),
            Header(text="Фейк подарки в профиле"),
            Switch(key="fake_gifts", text="Добавлять подарки в мой профиль", default=True),
            Input(key="gifts_count", text="Сколько подарков добавить", default="12"),
            Divider(text="Команды в личном чате:\n"
                         ".fgift — «отправить» случайный подарок\n"
                         ".fgift 3 — подарок №3 из каталога\n"
                         ".fgift in — как будто подарок прислали тебе"),
        ]

    # ---------- утилиты ----------

    def _int(self, key, default):
        try:
            return max(0, int(str(self.get_setting(key, str(default))).strip()))
        except Exception:
            return default

    def _catalog(self, account):
        try:
            ctrl = StarsController.getInstance(account)
            gifts = ctrl.gifts
            if gifts is None or gifts.size() == 0:
                ctrl.loadStarGifts()
                return []
            return [gifts.get(i) for i in range(gifts.size())]
        except Exception as e:
            log(f"[FakeGifts] catalog error: {e}")
            return []

    def _patch_balance(self, obj):
        fake = self._int("stars_amount", 99999)
        bal = obj.balance
        if isinstance(bal, int):  # старые версии: balance = long
            obj.balance = fake
            return True
        if "Ton" in bal.getClass().getSimpleName():  # TON-баланс не трогаем
            return False
        bal.amount = fake
        bal.nanos = 0
        return True

    # ---------- хуки запросов ----------

    def pre_request_hook(self, request_name, account, request):
        if "getSavedStarGifts" in request_name:
            mine = False
            try:
                peer = request.peer
                is_self = (peer.getClass().getSimpleName() == "TL_inputPeerSelf"
                           or peer.user_id == get_user_config().getClientUserId())
                mine = is_self and not request.offset  # только первая страница
            except Exception:
                pass
            self._pending.append(mine)
        return HookResult()

    def post_request_hook(self, request_name, account, response, error):
        try:
            if "getSavedStarGifts" in request_name:
                mine = self._pending.popleft() if self._pending else False
                if mine and response is not None and self.get_setting("fake_gifts", True):
                    if self._add_fake_gifts(account, response):
                        return HookResult(strategy=HookStrategy.MODIFY, response=response)

            elif response is None:
                pass

            elif "getStarsStatus" in request_name:
                if self.get_setting("fake_stars", True) and self._patch_balance(response):
                    return HookResult(strategy=HookStrategy.MODIFY, response=response)

            elif request_name == "TL_users_getFullUser":
                if self.get_setting("fake_gifts", True):
                    full = response.full_user
                    if full.id == get_user_config().getClientUserId():
                        full.stargifts_count += self._int("gifts_count", 12)
                        full.flags2 |= 256
                        return HookResult(strategy=HookStrategy.MODIFY, response=response)
        except Exception as e:
            log(f"[FakeGifts] post_request error ({request_name}): {e}")
        return HookResult()

    def on_update_hook(self, update_name, account, update):
        try:
            if update_name == "TL_updateStarsBalance" and self.get_setting("fake_stars", True):
                if self._patch_balance(update):
                    return HookResult(strategy=HookStrategy.MODIFY, update=update)
        except Exception as e:
            log(f"[FakeGifts] update error: {e}")
        return HookResult()

    # ---------- фейк список подарков ----------

    def _add_fake_gifts(self, account, response):
        catalog = self._catalog(account)
        n = self._int("gifts_count", 12)
        if not catalog or n == 0:
            return 0
        cls = getattr(TL_stars, "TL_savedStarGift", None) or TL_stars.SavedStarGift
        rnd = random.Random(1337)  # фиксированный seed — список одинаковый при каждом открытии
        now = int(time.time())
        for _ in range(n):
            gift = rnd.choice(catalog)
            saved = cls()
            saved.gift = gift
            saved.name_hidden = True  # «аноним», чтобы не подставлять реальных людей
            saved.date = now - rnd.randint(0, 90 * 86400)
            try:
                saved.convert_stars = gift.convert_stars
            except Exception:
                pass
            response.gifts.add(saved)
        response.count += n
        return n

    # ---------- фейк отправка подарка ----------

    def on_send_message_hook(self, account, params):
        text = (getattr(params, "message", None) or "").strip()
        parts = text.split()
        if not parts or parts[0] != ".fgift":
            return HookResult()

        args = parts[1:]
        incoming = "in" in args
        idx = next((int(a) for a in args if a.isdigit()), None)
        dialog_id = params.peer

        if dialog_id <= 0:
            BulletinHelper.show_error("Работает только в личных чатах")
            return HookResult(strategy=HookStrategy.CANCEL)

        catalog = self._catalog(account)
        if not catalog:
            BulletinHelper.show_info("Каталог подарков загружается, попробуй ещё раз")
            return HookResult(strategy=HookStrategy.CANCEL)

        if idx is not None and 1 <= idx <= len(catalog):
            gift = catalog[idx - 1]
        else:
            gift = random.choice(catalog)

        run_on_ui_thread(lambda: self._show_fake_gift(account, dialog_id, gift, incoming))
        return HookResult(strategy=HookStrategy.CANCEL)

    def _show_fake_gift(self, account, dialog_id, gift, incoming):
        try:
            me = get_user_config().getClientUserId()

            action = TLRPC.TL_messageActionStarGift()
            action.gift = gift
            try:
                action.convert_stars = gift.convert_stars
            except Exception:
                pass

            msg = TLRPC.TL_messageService()
            msg.id = get_user_config().getNewMessageId()
            msg.local_id = msg.id
            msg.dialog_id = dialog_id
            msg.date = int(time.time())
            msg.out = not incoming
            msg.unread = False
            msg.flags = 256

            from_peer = TLRPC.TL_peerUser()
            from_peer.user_id = dialog_id if incoming else me
            msg.from_id = from_peer

            to_peer = TLRPC.TL_peerUser()
            to_peer.user_id = dialog_id
            msg.peer_id = to_peer

            msg.action = action

            obj = MessageObject(account, msg, True, True)
            arr = ArrayList()
            arr.add(obj)

            mc = get_messages_controller()
            try:
                mc.updateInterfaceWithMessages(dialog_id, arr, 0)
            except Exception:
                mc.updateInterfaceWithMessages(dialog_id, arr, False)
        except Exception as e:
            log(f"[FakeGifts] show gift error: {e}")
            BulletinHelper.show_error(f"Ошибка: {e}")
