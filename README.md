# song-player-with-eyes
song player
import sys
import os
import cv2
import pygame
from PyQt5.QtWidgets import (
    QApplication, QWidget, QVBoxLayout, QHBoxLayout, QLabel,
    QListWidget, QPushButton, QSizePolicy, QStatusBar
)
from PyQt5.QtGui import QFont, QPalette, QColor, QImage, QPixmap
from PyQt5.QtCore import Qt, QThread, pyqtSignal

class VideoThread(QThread):
    change_pixmap_signal = pyqtSignal(QImage)
    eye_action_signal = pyqtSignal(str)

    def __init__(self):
        super().__init__()
        self._run_flag = True
        
        self.eye_cascade = cv2.CascadeClassifier(cv2.data.haarcascades + 'haarcascade_eye.xml')

    def run(self):
        cap = cv2.VideoCapture(0)
        blink_counter = 0

        while self._run_flag:
            ret, frame = cap.read()
            if not ret:
                continue

            gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
            eyes = self.eye_cascade.detectMultiScale(gray, 1.3, 5)

            if len(eyes) == 0:
                blink_counter += 1
            else:
                if blink_counter > 3:  # چند فریم بدون چشم → یعنی پلک زدی
                    self.eye_action_signal.emit("blink")
                blink_counter = 0

    
            rgb_image = cv2.cvtColor(frame, cv2.COLOR_BGR2RGB)
            h, w, ch = rgb_image.shape
            bytes_per_line = ch * w
            qt_image = QImage(rgb_image.data, w, h, bytes_per_line, QImage.Format_RGB888)
            self.change_pixmap_signal.emit(qt_image)

        cap.release()

    def stop(self):
        self._run_flag = False
        self.wait()

class MusicPlayer(QWidget):
    def __init__(self, songs):
        super().__init__()
        self.setWindowTitle("Eye Music Control")
        self.setFixedSize(900, 520)

        self.songs = songs
        self.current_index = 0
        self.is_playing = False

        pygame.mixer.init()

        font_main = QFont("Segoe UI", 11)
        font_buttons = QFont("Segoe UI", 14, QFont.Bold)
        self.setFont(font_main)

        palette = self.palette()
        palette.setColor(QPalette.Window, QColor(40, 44, 52))
        palette.setColor(QPalette.WindowText, Qt.white)
        self.setPalette(palette)

        main_layout = QHBoxLayout()
        main_layout.setContentsMargins(15, 15, 15, 15)
        main_layout.setSpacing(20)

    
        self.video_label = QLabel()
        self.video_label.setFixedSize(640, 480)
        self.video_label.setStyleSheet("border-radius: 10px; background-color: black;")
        main_layout.addWidget(self.video_label)

    
        right_layout = QVBoxLayout()
        right_layout.setSpacing(15)

        self.list_widget = QListWidget()
        self.list_widget.setStyleSheet("""
            QListWidget {
                background-color: #2d2f38;
                color: white;
                border-radius: 8px;
                padding: 5px;
            }
            QListWidget::item:selected {
                background-color: #3c78d8;
                color: white;
            }
        """)
        self.list_widget.setFont(font_main)
        for song in self.songs:
            self.list_widget.addItem(os.path.basename(song))
        self.list_widget.setCurrentRow(0)
        right_layout.addWidget(self.list_widget)

        
        self.play_pause_btn = QPushButton("▶️ Play")
        self.play_pause_btn.setFont(font_buttons)
        self.play_pause_btn.setStyleSheet("""
            QPushButton {
                background-color: #3c78d8;
                color: white;
                border-radius: 12px;
                padding: 12px 25px;
            }
            QPushButton:hover {
                background-color: #5593f3;
            }
        """)
        self.play_pause_btn.setSizePolicy(QSizePolicy.Expanding, QSizePolicy.Fixed)
        self.play_pause_btn.clicked.connect(self.play_pause)
        right_layout.addWidget(self.play_pause_btn)

    
        btn_layout = QHBoxLayout()
        self.prev_btn = QPushButton("⏮ Previous")
        self.next_btn = QPushButton("Next ⏭")
        for btn in (self.prev_btn, self.next_btn):
            btn.setFont(QFont("Segoe UI", 12, QFont.Bold))
            btn.setStyleSheet("""
                QPushButton {
                    background-color: #5a5f73;
                    color: white;
                    border-radius: 10px;
                    padding: 10px 18px;
                }
                QPushButton:hover {
                    background-color: #7b80a0;
                }
            """)
            btn.setSizePolicy(QSizePolicy.Expanding, QSizePolicy.Fixed)

        self.prev_btn.clicked.connect(self.previous_song)
        self.next_btn.clicked.connect(self.next_song)

        btn_layout.addWidget(self.prev_btn)
        btn_layout.addWidget(self.next_btn)
        right_layout.addLayout(btn_layout)

    
        self.status_bar = QStatusBar()
        self.status_bar.setStyleSheet("color: white; background-color: transparent;")
        right_layout.addWidget(self.status_bar)

        main_layout.addLayout(right_layout)
        self.setLayout(main_layout)

    
        self.thread = VideoThread()
        self.thread.change_pixmap_signal.connect(self.update_image)
        self.thread.eye_action_signal.connect(self.handle_eye_action)
        self.thread.start()

        self.list_widget.currentRowChanged.connect(self.change_song)

        self.update_status()


    def change_song(self, index):
        self.current_index = index
        if self.is_playing:
            self.play_song()

    def play_song(self):
        pygame.mixer.music.load(self.songs[self.current_index])
        pygame.mixer.music.play()
        self.is_playing = True
        self.play_pause_btn.setText("⏸ Pause")
        self.update_status()

    def play_pause(self):
        if self.is_playing:
            pygame.mixer.music.pause()
            self.is_playing = False
            self.play_pause_btn.setText("▶️ Play")
            self.update_status("Paused")
        else:
            if not pygame.mixer.music.get_busy():
                self.play_song()
            else:
                pygame.mixer.music.unpause()
                self.is_playing = True
                self.play_pause_btn.setText("⏸ Pause")
                self.update_status()

    def previous_song(self):
        self.current_index = (self.current_index - 1) % len(self.songs)
        self.list_widget.setCurrentRow(self.current_index)
        self.play_song()

    def next_song(self):
        self.current_index = (self.current_index + 1) % len(self.songs)
        self.list_widget.setCurrentRow(self.current_index)
        self.play_song()

    def update_status(self, msg=None):
        if msg is None:
            msg = f"Playing: {os.path.basename(self.songs[self.current_index])}" if self.is_playing else "Paused"
        self.status_bar.showMessage(msg)

    
    def update_image(self, qt_image):
        self.video_label.setPixmap(QPixmap.fromImage(qt_image))

    
    def handle_eye_action(self, action):
        if action == "blink":
            self.play_pause()


if __name__ == "__main__":
    songs = [
        r"C:\Users\Asus\Downloads\Moein - Halghe Tala [320].mp3",
        r"C:\Users\Asus\Downloads\Moein - Bi To Nemitoonam [320].mp3"  
    ]
    app = QApplication(sys.argv)
    player = MusicPlayer(songs)
    player.show()
    sys.exit(app.exec_())
