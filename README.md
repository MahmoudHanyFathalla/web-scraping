#                 #CmakeLists.txt#                      #

cmake_minimum_required(VERSION 3.5)

project(untitled VERSION 0.1 LANGUAGES CXX)

set(CMAKE_AUTOUIC ON)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Manually set VLC include and library directories

# In this set I uesd the source code of vlc player to reach include folder which has the[ VLC folder which has vlc.h]
set(VLC_INCLUDE_DIRS "C:/Users/hp/Downloads/vlc-3.0.20/vlc-3.0.20/include")

# In this set I uesd the windows vlc player folder to reach [ VLC folder which has libvlc.dll]
set(VLC_LIBRARY_DIRS "C:/Program Files/VideoLAN/VLC")

# Add VLC library directory to CMAKE_PREFIX_PATH
set(CMAKE_PREFIX_PATH ${CMAKE_PREFIX_PATH} ${VLC_LIBRARY_DIRS})

find_package(QT NAMES Qt6 Qt5 REQUIRED COMPONENTS Widgets LinguistTools)
find_package(Qt${QT_VERSION_MAJOR} REQUIRED COMPONENTS Widgets LinguistTools)

set(TS_FILES untitled_en_GB.ts)

set(PROJECT_SOURCES
    main.cpp
    mainwindow.cpp
    mainwindow.h
    mainwindow.ui
    ${TS_FILES}
)

if(${QT_VERSION_MAJOR} GREATER_EQUAL 6)
    qt_add_executable(untitled
        MANUAL_FINALIZATION
        ${PROJECT_SOURCES}
    )
    qt_create_translation(QM_FILES ${CMAKE_SOURCE_DIR} ${TS_FILES})
else()
    if(ANDROID)
        add_library(untitled SHARED
            ${PROJECT_SOURCES}
        )
    else()
        add_executable(untitled
            ${PROJECT_SOURCES}
        )
    endif()

    qt5_create_translation(QM_FILES ${CMAKE_SOURCE_DIR} ${TS_FILES})
endif()

target_link_libraries(untitled PRIVATE
    Qt${QT_VERSION_MAJOR}::Widgets
)

# Include VLC headers
target_include_directories(untitled PRIVATE ${VLC_INCLUDE_DIRS})

# Link against VLC libraries
target_link_directories(untitled PRIVATE ${VLC_LIBRARY_DIRS})

# Link against VLC library (update the name based on your actual library)
target_link_libraries(untitled PRIVATE ${VLC_LIBRARY_DIRS}/libvlc.dll)
# make sure to put libvlc.dll

if(${QT_VERSION} VERSION_LESS 6.1.0)
    set(BUNDLE_ID_OPTION MACOSX_BUNDLE_GUI_IDENTIFIER com.example.untitled)
endif()

set_target_properties(untitled PROPERTIES
    ${BUNDLE_ID_OPTION}
    MACOSX_BUNDLE_BUNDLE_VERSION ${PROJECT_VERSION}
    MACOSX_BUNDLE_SHORT_VERSION_STRING ${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}
    MACOSX_BUNDLE TRUE
    WIN32_EXECUTABLE TRUE
)

include(GNUInstallDirs)
install(TARGETS untitled
    BUNDLE DESTINATION .
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

if(QT_VERSION_MAJOR EQUAL 6)
    qt_finalize_executable(untitled)
endif()
### ################################################################### ###
#                         #     mainwindow.h    #                         #

#ifndef MAINWINDOW_H
#define MAINWINDOW_H

#include <QMainWindow>
#include <vlc/vlc.h>
#include <QFrame>


QT_BEGIN_NAMESPACE
namespace Ui {
class MainWindow;
}
QT_END_NAMESPACE

struct libvlc_instance_t;
struct libvlc_media_player_t;

class QFrame;

class MainWindow : public QMainWindow
{
    Q_OBJECT

public:
    MainWindow(QWidget *parent = nullptr);
    ~MainWindow();

private:
    Ui::MainWindow *ui;
    libvlc_instance_t *vlcInstance;
    libvlc_media_player_t *vlcMediaPlayer;
    QFrame *videoFrame;
};

#endif // MAINWINDOW_H

# ######################################################## #
#                    mainwindow.cpp                        #

#include "mainwindow.h"
#include "ui_mainwindow.h"
#include <QVBoxLayout>
#include <vlc/vlc.h>
#include <QFrame>
#include <QLineEdit>
#include <QPushButton>

MainWindow::MainWindow(QWidget *parent)
    : QMainWindow(parent), ui(new Ui::MainWindow)
{
    ui->setupUi(this);

# Create a Qt frame to hold the VLC video output
    videoFrame = new QFrame(this);
    QVBoxLayout *layout = new QVBoxLayout(ui->centralwidget);
    layout->addWidget(videoFrame);

# Create a line edit for entering the video path
    QLineEdit *videoPathLineEdit = new QLineEdit(this);
    videoPathLineEdit->setPlaceholderText("Enter video path...");
    layout->addWidget(videoPathLineEdit);

# Create a button to load the video
    QPushButton *loadVideoButton = new QPushButton("Load Video", this);
    layout->addWidget(loadVideoButton);

# Connect the button click to load the video
    connect(loadVideoButton, &QPushButton::clicked, [this, videoPathLineEdit]() {
# Load the video from the entered path
        const char *videoPath = videoPathLineEdit->text().toStdString().c_str();
        libvlc_media_t *vlcMedia = libvlc_media_new_path(vlcInstance, videoPath);
        libvlc_media_player_set_media(vlcMediaPlayer, vlcMedia);

# Play the video
        libvlc_media_player_play(vlcMediaPlayer);
    });

# Initialize libVLC
    vlcInstance = libvlc_new(0, nullptr);
    vlcMediaPlayer = libvlc_media_player_new(vlcInstance);

# Set up the VLC widget
    void *videoWidget = (void *)videoFrame->winId();
    libvlc_media_player_set_hwnd(vlcMediaPlayer, videoWidget);
}

MainWindow::~MainWindow()
{
# Release libVLC resources
    libvlc_media_player_stop(vlcMediaPlayer);
    libvlc_media_player_release(vlcMediaPlayer);
    libvlc_release(vlcInstance);

    delete ui;
}

# ####################################################################  #
#                     main.cpp                  #

#include "mainwindow.h"

#include <QApplication>
#include <QLocale>
#include <QTranslator>

int main(int argc, char *argv[])
{
    QApplication a(argc, argv);

    QTranslator translator;
    const QStringList uiLanguages = QLocale::system().uiLanguages();
    for (const QString &locale : uiLanguages) {
        const QString baseName = "untitled_" + QLocale(locale).name();
        if (translator.load(":/i18n/" + baseName)) {
            a.installTranslator(&translator);
            break;
        }
    }
    MainWindow w;
    w.show();
    return a.exec();
}
