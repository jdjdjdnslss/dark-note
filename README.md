# dark-note
my personal dark note app

# ==========================================
# DARK NOTES - برنامه یادداشت برداری
# ساخته شده با Python و Kivy
# قابل اجرا روی ویندوز و اندروید
# ==========================================

import os
import sqlite3
from datetime import datetime

from kivy.app import App
from kivy.metrics import dp, sp
from kivy.uix.boxlayout import BoxLayout
from kivy.uix.gridlayout import GridLayout
from kivy.uix.scrollview import ScrollView
from kivy.uix.button import Button
from kivy.uix.label import Label
from kivy.uix.textinput import TextInput
from kivy.uix.popup import Popup
from kivy.core.window import Window


# رنگ های اصلی برنامه
BG = (0.07, 0.08, 0.12, 1)
PANEL = (0.12, 0.14, 0.20, 1)
CARD = (0.17, 0.19, 0.27, 1)
BLUE = (0.34, 0.40, 0.85, 1)
WHITE = (0.94, 0.95, 1, 1)
GRAY = (0.65, 0.68, 0.77, 1)
RED = (0.75, 0.22, 0.25, 1)


class DarkNotes(App):

    def build(self):

        # تنظیم رنگ پس زمینه پنجره
        Window.clearcolor = BG

        # ساخت محل ذخیره اطلاعات برنامه
        self.db_path = os.path.join(
            self.user_data_dir, "notes.db"
        )

        # اتصال به پایگاه داده
        self.db = sqlite3.connect(self.db_path)

        # ساخت جدول یادداشت ها در صورت نبودن
        self.db.execute("""
            CREATE TABLE IF NOT EXISTS notes (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                title TEXT NOT NULL,
                body TEXT NOT NULL,
                date TEXT NOT NULL
            )
        """)

        self.db.commit()

        # ساخت صفحه اصلی
        self.main = BoxLayout(
            orientation="vertical",
            padding=dp(12),
            spacing=dp(10)
        )

        self.main.add_widget(self.header())

        # کادر جستجو
        self.search = TextInput(
            hint_text="جستجو در یادداشت ها...",
            size_hint_y=None,
            height=dp(48),
            multiline=False,
            background_color=PANEL,
            foreground_color=WHITE,
            hint_text_color=GRAY,
            cursor_color=WHITE,
            padding=dp(12),
            font_size=sp(16)
        )

        # با تغییر متن جستجو، لیست به روز می شود
        self.search.bind(text=self.refresh_notes)
        self.main.add_widget(self.search)

        # فضای نمایش یادداشت ها
        self.scroll = ScrollView()

        self.notes_list = GridLayout(
            cols=1,
            spacing=dp(9),
            size_hint_y=None,
            padding=dp(2)
        )

        # ارتفاع لیست متناسب با تعداد یادداشت ها
        self.notes_list.bind(
            minimum_height=self.notes_list.setter("height")
        )

        self.scroll.add_widget(self.notes_list)
        self.main.add_widget(self.scroll)

        # دکمه ساخت یادداشت جدید
        add_button = Button(
            text="+ یادداشت جدید",
            size_hint_y=None,
            height=dp(55),
            background_normal="",
            background_color=BLUE,
            color=WHITE,
            font_size=sp(18)
        )

        add_button.bind(on_release=self.new_note)
        self.main.add_widget(add_button)

        self.refresh_notes()

        return self.main

    # --------------------------------------
    # سربرگ برنامه
    # --------------------------------------
    def header(self):

        box = BoxLayout(
            orientation="vertical",
            size_hint_y=None,
            height=dp(75),
            spacing=dp(3)
        )

        title = Label(
            text="[b]DARK NOTES[/b]",
            markup=True,
            font_size=sp(27),
            color=WHITE,
            halign="left"
        )

        subtitle = Label(
            text="دفترچه یادداشت شخصی شما",
            font_size=sp(13),
            color=GRAY,
            halign="left"
        )

        box.add_widget(title)
        box.add_widget(subtitle)

        return box

    # --------------------------------------
    # دریافت یادداشت ها از پایگاه داده
    # --------------------------------------
    def get_notes(self, keyword=""):

        cursor = self.db.cursor()

        if keyword:
            # جستجو در عنوان و متن یادداشت
            cursor.execute("""
                SELECT id, title, body, date
                FROM notes
                WHERE title LIKE ? OR body LIKE ?
                ORDER BY id DESC
            """, (
                "%" + keyword + "%",
                "%" + keyword + "%"
            ))
        else:
            cursor.execute("""
                SELECT id, title, body, date
                FROM notes
                ORDER BY id DESC
            """)

        return cursor.fetchall()

    # --------------------------------------
    # نمایش و به روزرسانی لیست یادداشت ها
    # --------------------------------------
    def refresh_notes(self, *args):

        self.notes_list.clear_widgets()

        keyword = self.search.text.strip()

        notes = self.get_notes(keyword)

        if not notes:
            empty = Label(
                text="هنوز یادداشتی وجود ندارد!",
                color=GRAY,
                size_hint_y=None,
                height=dp(100),
                font_size=sp(16)
            )

            self.notes_list.add_widget(empty)
            return

        for note in notes:

            note_id, title, body, date = note

            # ساخت کارت برای هر یادداشت
            card = Button(
                text=(
                    "[b]" + title + "[/b]\n"
                    + body[:90].replace("\n", " ")
                    + "\n\n" + date
                ),
                markup=True,
                size_hint_y=None,
                height=dp(115),
                background_normal="",
                background_color=CARD,
                color=WHITE,
                font_size=sp(14),
                halign="right",
                valign="middle",
                padding=(dp(12), dp(8))
            )

            # اتصال کارت به صفحه ویرایش همان یادداشت
            card.bind(
                on_release=lambda instance, nid=note_id:
                self.edit_note(nid)
            )

            self.notes_list.add_widget(card)

    # --------------------------------------
    # ساخت یادداشت جدید
    # --------------------------------------
    def new_note(self, instance):

        self.show_editor()

    # --------------------------------------
    # ویرایش یادداشت موجود
    # --------------------------------------
    def edit_note(self, note_id):

        cursor = self.db.cursor()

        cursor.execute(
            "SELECT title, body FROM notes WHERE id=?",
            (note_id,)
        )

        note = cursor.fetchone()

        if note:
            self.show_editor(
                note_id,
                note[0],
                note[1]
            )

    # --------------------------------------
    # پنجره نوشتن و ویرایش یادداشت
    # --------------------------------------
    def show_editor(
        self,
        note_id=None,
        old_title="",
        old_body=""
    ):

        layout = BoxLayout(
            orientation="vertical",
            padding=dp(12),
            spacing=dp(10)
        )

        # ورودی عنوان یادداشت
        title_input = TextInput(
            text=old_title,
            hint_text="عنوان یادداشت...",
            multiline=False,
            size_hint_y=None,
            height=dp(48),
            background_color=CARD,
            foreground_color=WHITE,
            hint_text_color=GRAY,
            cursor_color=WHITE,
            font_size=sp(16),
            padding=dp(10)
        )

        # ورودی متن یادداشت
        body_input = TextInput(
            text=old_body,
            hint_text="متن یادداشت خود را بنویس...",
            multiline=True,
            background_color=CARD,
            foreground_color=WHITE,
            hint_text_color=GRAY,
            cursor_color=WHITE,
            font_size=sp(16),
            padding=dp(12)
        )

        layout.add_widget(title_input)
        layout.add_widget(body_input)

        # دکمه های پایین پنجره
        buttons = BoxLayout(
            size_hint_y=None,
            height=dp(50),
            spacing=dp(7)
        )

        save_button = Button(
            text="ذخیره",
            background_normal="",
            background_color=BLUE
        )

        close_button = Button(
            text="بستن",
            background_normal="",
            background_color=CARD
        )

        buttons.add_widget(save_button)
        buttons.add_widget(close_button)

        # اگر یادداشت قبلی باشد، دکمه حذف هم نمایش داده می شود
        delete_button = None

        if note_id is not None:

            delete_button = Button(
                text="حذف",
                background_normal="",
                background_color=RED
            )

            buttons.add_widget(delete_button)

        layout.add_widget(buttons)

        popup = Popup(
            title="یادداشت من",
            content=layout,
            size_hint=(0.94, 0.85),
            background_color=PANEL,
            title_color=WHITE,
            separator_color=BLUE,
            auto_dismiss=False
        )

        # ذخیره یادداشت در پایگاه داده
        def save_note(instance):

            title = title_input.text.strip()
            body = body_input.text.strip()

            # جلوگیری از ذخیره یادداشت خالی
            if not title and not body:
                return

            if not title:
                title = body.split("\n")[0][:40]

            now = datetime.now().strftime(
                "%Y/%m/%d - %H:%M"
            )

            if note_id is None:

                self.db.execute("""
                    INSERT INTO notes (title, body, date)
                    VALUES (?, ?, ?)
                """, (title, body, now))

            else:

                self.db.execute("""
                    UPDATE notes
                    SET title=?, body=?, date=?
                    WHERE id=?
                """, (title, body, now, note_id))

            self.db.commit()

            popup.dismiss()
            self.refresh_notes()

        # حذف یادداشت انتخاب شده
        def delete_note(instance):

            self.db.execute(
                "DELETE FROM notes WHERE id=?",
                (note_id,)
            )

            self.db.commit()

            popup.dismiss()
            self.refresh_notes()

        save_button.bind(on_release=save_note)
        close_button.bind(on_release=popup.dismiss)

        if delete_button:
            delete_button.bind(on_release=delete_note)

        popup.open()

    # --------------------------------------
    # بستن اتصال پایگاه داده
    # --------------------------------------
    def on_stop(self):

        if hasattr(self, "db"):
            self.db.close()


# اجرای برنامه
if __name__ == "__main__":
    DarkNotes().run()
    
