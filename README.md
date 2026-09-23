 private
 // Area Selection variables
    QRect m_selectedRect;
    bool m_useAreaSelection = false;
//methods
void RecordReplay::on_pushButton_fullscreen_clicked()
{
    m_useAreaSelection = false;
    m_selectedRect = QRect();

    ui->pushButton_fullscreen->setChecked(true);
    ui->pushButton_customarea->setChecked(false);
    qDebug() << "Recording mode switched to: Fullscreen";
}

void RecordReplay::on_pushButton_customarea_clicked()
{
    ui->pushButton_fullscreen->setChecked(false);
    ui->pushButton_customarea->setChecked(true);

    this->hide(); // Hide main window to allow clean area selection

    AreaSelectorWidget *selector = new AreaSelectorWidget();
    connect(selector, &AreaSelectorWidget::areaSelected, this, [=](const QRect &rect){
        if (rect.width() > 10 && rect.height() > 10) {
            // Force dimensions to even numbers for libx264/yuv420p compatibility
            int w = rect.width() & ~1;
            int h = rect.height() & ~1;

            m_selectedRect = QRect(rect.x(), rect.y(), w, h);
            m_useAreaSelection = true;

            qDebug() << "Custom area selected (Even pixels):" << m_selectedRect;
        } else {
            // Revert back to fullscreen if canceled or too small
            m_useAreaSelection = false;
            ui->pushButton_fullscreen->setChecked(true);
            ui->pushButton_customarea->setChecked(false);
            qDebug() << "Selection canceled or too small. Defaulted to Fullscreen.";
        }
        this->show(); // Bring main window back
    });

    //constructor

     // Make buttons act as mode indicators
        ui->pushButton_fullscreen->setCheckable(true);
        ui->pushButton_customarea->setCheckable(true);
        ui->pushButton_fullscreen->setChecked(true); // Default to fullscreen
        m_useAreaSelection = false;



    //.h

    #ifndef AREASELECTORWIDGET_H
#define AREASELECTORWIDGET_H

#include <QWidget>
#include <QRect>
#include <QPoint>

class QRubberBand;

class AreaSelectorWidget : public QWidget
{
    Q_OBJECT
public:
    explicit AreaSelectorWidget(QWidget *parent = nullptr);

signals:
    void areaSelected(const QRect &rect);

protected:
    void mousePressEvent(QMouseEvent *event) override;
    void mouseMoveEvent(QMouseEvent *event) override;
    void mouseReleaseEvent(QMouseEvent *event) override;
    void paintEvent(QPaintEvent *event) override;
    void keyPressEvent(QKeyEvent *event) override;

private:
    enum DragMode { None, ResizeTL, ResizeTC, ResizeTR, ResizeRC, ResizeBR, ResizeBC, ResizeBL, ResizeLC, NewSelect };

    QPoint m_origin;      // Anchor point (opposite side/corner)
    QRubberBand *m_rubberBand = nullptr;
    bool m_isSelecting = false;
    bool m_hasSelection = false;
    DragMode m_dragMode = None;

    DragMode getDragModeAt(const QPoint &pos) const;
    void updateCursorForPosition(const QPoint &pos);
};

#endif // AREASELECTORWIDGET_H
//.cpp

#include "areaselectorwidget.h"
#include <QRubberBand>
#include <QMouseEvent>
#include <QPainter>
#include <QScreen>
#include <QGuiApplication>
#include <QKeyEvent>

AreaSelectorWidget::AreaSelectorWidget(QWidget *parent) : QWidget(parent)
{
    setWindowFlags(Qt::FramelessWindowHint | Qt::WindowStaysOnTopHint | Qt::Tool);
    setAttribute(Qt::WA_TranslucentBackground);
    setMouseTracking(true); // Required for real-time hover cursor updates
    setCursor(Qt::CrossCursor);

    if (QScreen *screen = QGuiApplication::primaryScreen()) {
        setGeometry(screen->geometry());
    }
}

AreaSelectorWidget::DragMode AreaSelectorWidget::getDragModeAt(const QPoint &p) const
{
    if (!m_hasSelection || !m_rubberBand) return NewSelect;

    QRect r = m_rubberBand->geometry();
    int tolerance = 12; // Hit-box radius for handles

    QPoint tl = r.topLeft();
    QPoint tc(r.center().x(), r.top());
    QPoint tr = r.topRight();
    QPoint rc(r.right(), r.center().y());
    QPoint br = r.bottomRight();
    QPoint bc(r.center().x(), r.bottom());
    QPoint bl = r.bottomLeft();
    QPoint lc(r.left(), r.center().y());

    if (QLineF(p, tl).length() <= tolerance) return ResizeTL;
    if (QLineF(p, tr).length() <= tolerance) return ResizeTR;
    if (QLineF(p, bl).length() <= tolerance) return ResizeBL;
    if (QLineF(p, br).length() <= tolerance) return ResizeBR;
    if (QLineF(p, tc).length() <= tolerance) return ResizeTC;
    if (QLineF(p, bc).length() <= tolerance) return ResizeBC;
    if (QLineF(p, lc).length() <= tolerance) return ResizeLC;
    if (QLineF(p, rc).length() <= tolerance) return ResizeRC;

    if (r.contains(p)) return None; // Inside box, but not on handle

    return NewSelect; // Clicked far outside, draw new box
}

void AreaSelectorWidget::updateCursorForPosition(const QPoint &p)
{
    if (!m_hasSelection) {
        setCursor(Qt::CrossCursor);
        return;
    }

    DragMode mode = getDragModeAt(p);
    switch (mode) {
        case ResizeTL:
        case ResizeBR:
            setCursor(Qt::SizeFDiagCursor);
            break;
        case ResizeTR:
        case ResizeBL:
            setCursor(Qt::SizeBDiagCursor);
            break;
        case ResizeTC:
        case ResizeBC:
            setCursor(Qt::SizeVerCursor);
            break;
        case ResizeLC:
        case ResizeRC:
            setCursor(Qt::SizeHorCursor);
            break;
        default:
            setCursor(Qt::ArrowCursor);
            break;
    }
}

void AreaSelectorWidget::mousePressEvent(QMouseEvent *event)
{
    if (event->button() == Qt::LeftButton) {
        QPoint p = event->pos();
        m_dragMode = getDragModeAt(p);

        if (m_dragMode == NewSelect || !m_hasSelection) {
            m_origin = p;
            if (!m_rubberBand) {
                m_rubberBand = new QRubberBand(QRubberBand::Rectangle, this);
            }
            m_rubberBand->setGeometry(QRect(m_origin, QSize()));
            m_rubberBand->show();
            m_hasSelection = false;
        } else if (m_hasSelection && m_rubberBand) {
            QRect r = m_rubberBand->geometry();
            switch (m_dragMode) {
                case ResizeTL: m_origin = r.bottomRight(); break;
                case ResizeTR: m_origin = r.bottomLeft();  break;
                case ResizeBL: m_origin = r.topRight();    break;
                case ResizeBR: m_origin = r.topLeft();     break;
                case ResizeTC: m_origin = r.bottomLeft();  break;
                case ResizeBC: m_origin = r.topLeft();     break;
                case ResizeLC: m_origin = r.topRight();    break;
                case ResizeRC: m_origin = r.topLeft();     break;
                default: break;
            }
        }
        m_isSelecting = true;
    }
}

void AreaSelectorWidget::mouseMoveEvent(QMouseEvent *event)
{
    QPoint p = event->pos();
    if (!m_isSelecting) {
        updateCursorForPosition(p);
    } else if (m_rubberBand) {
        QRect r = m_rubberBand->geometry();

        if (m_dragMode == ResizeTC) {
            m_rubberBand->setGeometry(QRect(QPoint(r.left(), p.y()), QPoint(r.right(), r.bottom())).normalized());
        } else if (m_dragMode == ResizeBC) {
            m_rubberBand->setGeometry(QRect(QPoint(r.left(), r.top()), QPoint(r.right(), p.y())).normalized());
        } else if (m_dragMode == ResizeLC) {
            m_rubberBand->setGeometry(QRect(QPoint(p.x(), r.top()), QPoint(r.right(), r.bottom())).normalized());
        } else if (m_dragMode == ResizeRC) {
            m_rubberBand->setGeometry(QRect(QPoint(r.left(), r.top()), QPoint(p.x(), r.bottom())).normalized());
        } else {
            m_rubberBand->setGeometry(QRect(m_origin, p).normalized());
        }
        update();
    }
}

void AreaSelectorWidget::mouseReleaseEvent(QMouseEvent *event)
{
    if (event->button() == Qt::LeftButton && m_isSelecting) {
        m_isSelecting = false;
        if (m_rubberBand && m_rubberBand->geometry().width() > 10 && m_rubberBand->geometry().height() > 10) {
            m_hasSelection = true;
        }
        updateCursorForPosition(event->pos());
        update();
    }
}

void AreaSelectorWidget::paintEvent(QPaintEvent *)
{
    QPainter painter(this);
    painter.fillRect(rect(), QColor(0, 0, 0, 80)); // Dark overlay tint

    if (m_rubberBand && m_rubberBand->isVisible()) {
        QRect r = m_rubberBand->geometry();

        // Draw selection frame
        painter.setPen(QPen(Qt::cyan, 2, Qt::SolidLine));
        painter.setBrush(Qt::NoBrush);
        painter.drawRect(r);

        // Draw 8 interactive extender handles
        int handleSize = 8;
        painter.setBrush(Qt::white);
        painter.setPen(QPen(Qt::black, 1));

        QPoint points[8] = {
            r.topLeft(),
            QPoint(r.center().x(), r.top()),
            r.topRight(),
            QPoint(r.right(), r.center().y()),
            r.bottomRight(),
            QPoint(r.center().x(), r.bottom()),
            r.bottomLeft(),
            QPoint(r.left(), r.center().y())
        };

        for (int i = 0; i < 8; ++i) {
            painter.drawRect(points[i].x() - handleSize / 2, points[i].y() - handleSize / 2, handleSize, handleSize);
        }
    }

    // Instructional text
    painter.setPen(Qt::white);
    painter.setFont(QFont("Arial", 12, QFont::Bold));
    painter.drawText(rect(), Qt::AlignTop | Qt::AlignHCenter,
                     "\nDrag handles to extend/resize area. Press [ENTER] to Confirm or [ESC] to Cancel.");
}

void AreaSelectorWidget::keyPressEvent(QKeyEvent *event)
{
    if (event->key() == Qt::Key_Return || event->key() == Qt::Key_Enter) {
        if (m_hasSelection && m_rubberBand) {
            QRect rect = m_rubberBand->geometry();
            emit areaSelected(rect);
            close();
            deleteLater();
        }
    }
    else if (event->key() == Qt::Key_Escape) {
        emit areaSelected(QRect());
        close();
        deleteLater();
    }
}
    selector->showFullScreen();
}







QStringList RecordReplay::getFFmpegArguments(const QString &outputPath)
{
    QStringList arguments;

    #ifdef Q_OS_WIN
        arguments << "-framerate" << "30";

        // Pass custom area width, height, and offsets to FFmpeg if custom mode is enabled
        if (m_useAreaSelection && m_selectedRect.isValid() && m_selectedRect.width() > 0 && m_selectedRect.height() > 0) {
            arguments << "-video_size" << QString("%1x%2").arg(m_selectedRect.width()).arg(m_selectedRect.height())
                      << "-offset_x" << QString::number(m_selectedRect.x())
                      << "-offset_y" << QString::number(m_selectedRect.y());
        }

        arguments << "-f" << "gdigrab"
                << "-draw_mouse" << "1"
                  << "-i" << "desktop"
                  << "-c:v" << "libx264"
                  << "-preset" << "veryfast"
                  << "-crf" << "18"
                  << "-pix_fmt" << "yuv420p"
                  << "-y"
                  << outputPath;
    #endif

        return arguments;
}  header//#include <areaselectorwidget.h>
