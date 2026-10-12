<img width="1254" height="700" alt="image" src="https://github.com/user-attachments/assets/a3e66112-eb8a-450c-8039-edf4c54db32d" />
<img width="1275" height="709" alt="image" src="https://github.com/user-attachments/assets/1a01f7aa-f945-4eac-8b44-62a220a6426c" />

from abc import ABC, abstractmethod
from datetime import datetime
from cryptography.fernet import Fernet
import flet as ft


class CustomMapStorage:

    def __init__(self):
        self._keys = []
        self._values = []

    @staticmethod
    def _validate_key(key):
        if not isinstance(key, str) or not key.strip():
            raise ValueError("Ключ должен быть непустой строкой")

    def put(self, key, value):
        self._validate_key(key)
        if key in self._keys:
            idx = self._keys.index(key)
            self._values[idx] = value
        else:
            self._keys.append(key)
            self._values.append(value)

    def get(self, key):
        self._validate_key(key)
        if key in self._keys:
            return self._values[self._keys.index(key)]
        return None

    def get_all_items(self):
        return list(zip(self._keys, self._values))

    def __len__(self):
        return len(self._keys)


class AbstractEntity(ABC):

    @abstractmethod
    def get_summary(self) -> str:
        pass


class PasswordRecord(AbstractEntity):

    def __init__(self, service: str, login: str, encrypted_pass: str):
        self._service = service
        self._login = login
        self._encrypted_pass = encrypted_pass

    @property
    def service(self) -> str:
        return self._service

    @service.setter
    def service(self, value: str):
        if value:
            self._service = value

    @property
    def login(self) -> str:
        return self._login

    @login.setter
    def login(self, value: str):
        if value:
            self._login = value

    @property
    def encrypted_pass(self) -> str:
        return self._encrypted_pass

    @encrypted_pass.setter
    def encrypted_pass(self, value: str):
        self._encrypted_pass = value

    def get_summary(self) -> str:
        return f"Сервис: {self._service} | Логин: {self._login}"

    def __str__(self) -> str:
        return f"🔐 {self._service} ({self._login})"

    def __repr__(self) -> str:
        return f"PasswordRecord(service='{self._service}', login='{self._login}')"

    def __eq__(self, other) -> bool:
        if isinstance(other, PasswordRecord):
            return (
                self._service.lower() == other._service.lower()
                and self._login.lower() == other._login.lower()
            )
        return False

    def __lt__(self, other) -> bool:
        return self._service.lower() < other._service.lower()

    def __gt__(self, other) -> bool:
        return self._service.lower() > other._service.lower()


class StyledTextField(ft.TextField):

    def __init__(
        self, label: str, password: bool = False, can_reveal: bool = False
    ):
        super().__init__()
        self.label = label
        self.password = password
        self.can_reveal_password = can_reveal
        self.width = 380
        self.border_radius = 8
        self.filled = True


class PrimaryButton(ft.ElevatedButton):

    def __init__(self, text: str, on_click_func):
        super().__init__()
        self.text = text
        self.on_click = on_click_func
        self.bgcolor = ft.colors.PINK_700
        self.color = ft.colors.WHITE
        self.width = 380
        self.height = 45


class TitleText(ft.Text):

    def __init__(self, value: str):
        super().__init__(
            value=value, size=22, weight=ft.FontWeight.BOLD, color=ft.colors.PINK_900
        )


class SubtitleText(ft.Text):

    def __init__(self, value: str):
        super().__init__(value=value, size=14, color=ft.colors.GREY_700)


class CustomDivider(ft.Divider):

    def __init__(self):
        super().__init__(height=20, color=ft.colors.PINK_200)


class AppCardContainer(ft.Container):

    def __init__(self, content_widget):
        super().__init__()
        self.content = content_widget
        self.padding = 15
        self.border_radius = 12
        self.bgcolor = ft.colors.WHITE
        self.shadow = ft.BoxShadow(
            spread_radius=1,
            blur_radius=10,
            color=ft.colors.PINK_200,
            offset=ft.Offset(0, 4),
        )


class MainColumn(ft.Column):

    def __init__(self, controls_list):
        super().__init__()
        self.controls = controls_list
        self.alignment = ft.MainAxisAlignment.CENTER
        self.horizontal_alignment = ft.CrossAxisAlignment.CENTER
        self.spacing = 15


class AccountsListView(ft.ListView):

    def __init__(self):
        super().__init__()
        self.expand = 1
        self.spacing = 8
        self.padding = 10
        self.auto_scroll = True


class AccountItemRow(ft.Row):

    def __init__(self, service: str, login: str, password: str):
        super().__init__()
        self.service = service
        self.login = login
        self.password = password
        self.is_visible = False

        self.pass_text = ft.Text(
            f"🔐 {self.service} | {self.login} | ••••••", size=13
        )

        self.btn_toggle = ft.IconButton(
            icon=ft.icons.VISIBILITY_OFF,
            icon_size=18,
            tooltip="Показать / скрыть пароль",
            on_click=self.toggle_password,
        )

        self.controls = [
            ft.Row([self.pass_text], expand=True),
            self.btn_toggle,
        ]
        self.alignment = ft.MainAxisAlignment.SPACE_BETWEEN
        self.vertical_alignment = ft.CrossAxisAlignment.CENTER

    def toggle_password(self, e):
        self.is_visible = not self.is_visible
        if self.is_visible:
            self.pass_text.value = f"🔓 {self.service} | {self.login} | {self.password}"
            self.btn_toggle.icon = ft.icons.VISIBILITY
        else:
            self.pass_text.value = f"🔐 {self.service} | {self.login} | ••••••"
            self.btn_toggle.icon = ft.icons.VISIBILITY_OFF
        self.update()


def main(page: ft.Page):
    time_in = datetime.now().strftime("%d.%m.%Y %H:%M:%S")
    username = "StudentExaminer"

    page.title = "PassManager — Secure Vault"
    page.vertical_alignment = ft.MainAxisAlignment.CENTER
    page.horizontal_alignment = ft.CrossAxisAlignment.CENTER
    page.window_width = 460
    page.window_height = 720

    key = Fernet.generate_key()
    cipher = Fernet(key)

    storage = CustomMapStorage()

    initial_services = [
        ("Google", "1"),
        ("GitHub", "2"),
        ("VK", "3"),
        ("Tg", "4"),
        ("Habr", "5"),
        ("x", "6"),
        ("Discord", "7"),
        ("Yandex", "8"),
        ("Steam", "9"),
        ("Notion", "10"),
    ]

    for s_name, s_login in initial_services:
        enc_pw = cipher.encrypt(b"123!").decode()
        rec = PasswordRecord(s_name, s_login, enc_pw)
        storage.put(s_name, rec)

    title = TitleText("Менеджер паролей")
    subtitle = SubtitleText(
        f"Загружено записей из базы: {len(storage.get_all_items())}"
    )

    inp_service = StyledTextField(label="Название сервиса (например, Twitter)")
    inp_login = StyledTextField(label="Логин или Email")
    inp_password = StyledTextField(label="Пароль", password=True, can_reveal=True)

    output_list = AccountsListView()

    def refresh_list():
        output_list.controls.clear()
        for s_name, record in storage.get_all_items():
            try:
                decrypted_password = cipher.decrypt(
                    record.encrypted_pass.encode()
                ).decode()
            except Exception:
                decrypted_password = "Ошибка"

            output_list.controls.append(
                AccountItemRow(record.service, record.login, decrypted_password)
            )
        page.update()

    refresh_list()

    def handle_add(e):
        if not inp_service.value or not inp_login.value or not inp_password.value:
            return

        encrypted_val = cipher.encrypt(inp_password.value.encode()).decode()
        new_record = PasswordRecord(
            inp_service.value, inp_login.value, encrypted_val
        )
        storage.put(inp_service.value, new_record)

        inp_service.value = ""
        inp_login.value = ""
        inp_password.value = ""

        subtitle.value = f"Загружено записей из базы: {len(storage.get_all_items())}"
        refresh_list()

    btn_save = PrimaryButton(text="Добавить пароль", on_click_func=handle_add)

    list_container = AppCardContainer(
        ft.Column(
            [
                ft.Text(
                    "Сохраненные аккаунты:",
                    weight=ft.FontWeight.BOLD,
                    size=14,
                ),
                ft.Container(content=output_list, height=220, width=380),
            ]
        )
    )

    page.add(
        MainColumn([
            title,
            subtitle,
            CustomDivider(),
            inp_service,
            inp_login,
            inp_password,
            btn_save,
            CustomDivider(),
            list_container,
        ])
    )

    def on_close(e):
        time_out = datetime.now().strftime("%d.%m.%Y %H:%M:%S")
        report_content = f"""+-------------------------------------------------------------+
| login | time_in | time_out |
{username} | {time_in} | {time_out}
+-------------------------------------------------------------+
"""
        with open("app_report.txt", "w", encoding="utf-8") as f:
            f.write(report_content)
        page.window_destroy()

    page.window_on_close = on_close


if __name__ == "__main__":
    ft.app(target=main)
