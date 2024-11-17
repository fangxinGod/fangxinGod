- 👋 Hi, I’m @fangxinGod
- 👀 I’m interested in ...
- 🌱 I’m currently learning ...
- 💞️ I’m looking to collaborate on ...
- 📫 How to reach me ...
- 😄 Pronouns: ...
- ⚡ Fun fact: ...

<!---
fangxinGod/fangxinGod is a ✨ special ✨ repository because its `README.md` (this file) appears on your GitHub profile.
You can click the Preview link to take a look at your changes.


#include <QApplication>
#include <QMainWindow>
#include <QPushButton>
#include <QVBoxLayout>
#include <QHBoxLayout>
#include <QFileDialog>
#include <QLabel>
#include <QMessageBox>
#include <QPixmap>
#include <QDebug>
#include <QListWidget>
#include <QFile>
#include <QTextStream>
#include <QDir>
#include <QMenuBar>
#include <QAction>
#include <QRegularExpression>

class SimpleAnnotationTool : public QMainWindow {
    Q_OBJECT

public:
    SimpleAnnotationTool(QWidget* parent = nullptr) : QMainWindow(parent), currentIndex(-1) {
        setupUI();
    }

private slots:
    void loadImages() {
        QString folder = QFileDialog::getExistingDirectory(this, tr("select image menu"));
        if (folder.isEmpty()) {
            QMessageBox::warning(this, tr("error"), tr("not select any menu"));
            return;
        }

        QDir directory(folder);
        QStringList filters;
        filters << "*.png" << "*.jpg" << "*.bmp";
        imageFiles = directory.entryList(filters, QDir::Files);
        imageDir = folder;

        if (imageFiles.isEmpty()) {
            QMessageBox::warning(this, tr("warning"), tr("not use file in this menu"));
            return;
        }

        currentIndex = 0;
        loadCurrentImage();
    }

    void loadCurrentImage() {
        if (currentIndex >= 0 && currentIndex < imageFiles.size()) {
            imageLabel->clear();

            QString imagePath = QDir::toNativeSeparators(imageDir + "/" + imageFiles[currentIndex]);
            qDebug() << "loading image path：" << imagePath;

            QImage image(imagePath);
            if (!image.isNull()) {
                imageLabel->setPixmap(QPixmap::fromImage(image).scaled(imageLabel->size(), Qt::KeepAspectRatio, Qt::SmoothTransformation));
                annotations.clear();
                annotationList->clear();

                loadAnnotationsFromCSV();

                enableAnnotationButtons(true);
                enableNavigationButtons(true);

                imageInfoLabel->setText(tr("Image:%1(%2/%3)").arg(imageFiles[currentIndex]).arg(currentIndex+1).arg(imageFiles.size()));

                int markedCount = 0;
                int unmarkedCount = 0;
                for (const QString& fileName : imageFiles) {
                    if (isImageMarked(fileName)) {
                        markedCount++;
                    } else {
                        unmarkedCount++;
                    }
                }


                imageStatusLabel->setText(tr("Marked: %1 | Unmarked: %2").arg(markedCount).arg(unmarkedCount));
            } else {
                QMessageBox::warning(this, tr("error"), tr("not load image：") + imagePath);
                enableNavigationButtons(false);
            }
        }
    }
    bool isImageMarked(const QString& fileName) {
        QFile file("/Users/xiaohou/Desktop/annotations.csv");
        if (!file.open(QIODevice::ReadOnly | QIODevice::Text)) {
            return false;
        }

        QTextStream stream(&file);
        bool marked = false;
        while (!stream.atEnd()) {
            QString line = stream.readLine();
            QStringList parts = line.split(",");
            if (!parts.isEmpty() && parts[0] == fileName) {
                marked = parts.mid(1).contains("1");
                break;
            }
        }
        file.close();
        return marked;
    }
    void loadAnnotationsFromCSV() {
        QString savePath = "/Users/xiaohou/Desktop/annotations.csv";
        QFile file(savePath);

        if (!file.exists()) {
            qDebug() << "not add label,because csv file not being";
            return;
        }

        if (!file.open(QIODevice::ReadOnly | QIODevice::Text)) {
            QMessageBox::warning(this, tr("error"), tr("not open label file！"));
            return;
        }

        QTextStream stream(&file);
        QStringList header;
        QString imageName = imageFiles[currentIndex];
        QStringList fileContent;

        while (!stream.atEnd()) {
            //QString line=stream.readLine();
            fileContent.append(stream.readLine());
        }

        if (!fileContent.isEmpty()) {
            header = fileContent[0].split(",");
        }

        for (int i = 1; i < fileContent.size(); ++i) {
            QStringList lineData = fileContent[i].split(",");
            if (lineData[0] == imageName) {
                for (int j = 1; j < lineData.size(); ++j) {
                    if (lineData[j] == "1") {
                        annotations.append(header[j]);
                        annotationList->addItem(header[j]);
                    }
                }
                break;
            }
        }

        file.close();
    }


    void enableNavigationButtons(bool enable){
        prevButton->setEnabled(enable);
        nextButton->setEnabled(enable);
    }

    void nextImage() {
        if (currentIndex < imageFiles.size() - 1) {
            currentIndex++;
            loadCurrentImage();
        } else {
            QMessageBox::information(this, tr("warnign"), tr("already is last image"));
        }
    }

    void previousImage() {
        if (currentIndex > 0) {
            currentIndex--;
            loadCurrentImage();
        } else {
            QMessageBox::information(this, tr("warning"), tr("already is first image"));
        }
    }

    void annotate(const QString& label) {
        if (!annotations.contains(label)) {
            annotations.append(label);
            annotationList->addItem(label);
            annotationsModified=true;
        }
    }

    void deleteAnnotation(QListWidgetItem* item) {
        QString label = item->text();
        annotations.removeAll(label);
        delete item;
        annotationsModified=true;
        qDebug() << "Annotation removed:" << label;
    }

    void saveAnnotations() {
        if(!annotationsModified){
            QMessageBox::information(this,tr("warning"),tr("not have any change need save"));
            return;
        }

        saveCurrentAnnotations();
        annotationsModified=false;
        QMessageBox::information(this, tr("success"), tr("label already save successfully"));
    }

    void saveCurrentAnnotations() {
        if (currentIndex < 0 || currentIndex >= imageFiles.size()) {
            return;
        }

        QString savePath = "/Users/xiaohou/Desktop/annotations.csv";
        QFile file(savePath);

        bool fileExists = QFile::exists(savePath);

        if (!file.open(QIODevice::ReadWrite | QIODevice::Text)) {
            QMessageBox::warning(this, tr("error"), tr("Unable to open file for saving annotations"));
            return;
        }

        QTextStream stream(&file);

        if (!fileExists) {
            QStringList header;
            header << "image name";
            header.append(allLabels);
            stream << header.join(",") << "\n";
        }

        file.seek(0);
        QStringList existingLines;
        bool found = false;
        int lineIndex = -1;

        while (!stream.atEnd()) {
            QString line = stream.readLine();
            existingLines.append(line);

            if (line.startsWith(imageFiles[currentIndex])) {
                found = true;
                lineIndex = existingLines.size() - 1;
            }
        }

        QString imageName = imageFiles[currentIndex];
        QStringList newLine;
        newLine << imageName;


        for (const QString& label : allLabels) {
            if (annotations.contains(label)) {
                newLine << "1";
            } else {
                newLine << "0";
            }
        }


        if (found && lineIndex >= 0) {
            existingLines[lineIndex] = newLine.join(",");
        } else {

            existingLines.append(newLine.join(","));
        }

        auto naturalSort = [](const QString& a, const QString& b) {
            QRegularExpression re("(\\d+)");
            auto ait = re.globalMatch(a);
            auto bit = re.globalMatch(b);

            while (ait.hasNext() && bit.hasNext()) {
                auto aMatch = ait.next();
                auto bMatch = bit.next();

                bool aIsNumber, bIsNumber;
                int aNum = aMatch.captured().toInt(&aIsNumber);
                int bNum = bMatch.captured().toInt(&bIsNumber);

                if (aIsNumber && bIsNumber) {
                    if (aNum != bNum) return aNum < bNum;
                } else {
                    if (aMatch.captured() != bMatch.captured()) {
                        return aMatch.captured() < bMatch.captured();
                    }
                }
            }
            return a < b;
        };

        std::sort(existingLines.begin()+1,existingLines.end(),naturalSort);

        file.resize(0);
        for (const QString& line : existingLines) {
            stream << line << "\n";
        }

        file.close();
        annotationsModified=false;

    }




private:
    void setupUI() {
        QWidget* centralWidget = new QWidget(this);
        setCentralWidget(centralWidget);

        QVBoxLayout* mainLayout = new QVBoxLayout(centralWidget);
        mainLayout->setContentsMargins(10, 10, 10, 10);
        mainLayout->setSpacing(15);


        QMenuBar* menuBar = new QMenuBar(this);
        setMenuBar(menuBar);
        QMenu* fileMenu = menuBar->addMenu(tr("file"));
        QAction* loadAction = fileMenu->addAction(tr("load image menu"));
        connect(loadAction, &QAction::triggered, this, &SimpleAnnotationTool::loadImages);

        QHBoxLayout* upperLayout = new QHBoxLayout;
        upperLayout->setSpacing(10);

        annotationList = new QListWidget(this);
        annotationList->setFixedWidth(150);
        connect(annotationList, &QListWidget::itemDoubleClicked, this, &SimpleAnnotationTool::deleteAnnotation);
        upperLayout->addWidget(annotationList);

        QVBoxLayout* imageLayout = new QVBoxLayout;
        imageLayout->setSpacing(2);

        imageStatusLabel = new QLabel(this);
        imageStatusLabel->setAlignment(Qt::AlignCenter);
        imageStatusLabel->setStyleSheet("font-size: 14px; color: gray; padding: 2px;");
        imageStatusLabel->setFixedHeight(20);

        imageLayout->addWidget(imageStatusLabel);

        imageInfoLabel = new QLabel(this);
        imageInfoLabel->setAlignment(Qt::AlignCenter);
        imageInfoLabel->setStyleSheet("font-size: 16px; color: black; padding: 2px;");
        imageInfoLabel->setFixedHeight(25);

        imageLayout->addWidget(imageInfoLabel);

        imageLabel = new QLabel(this);
        imageLabel->setFixedSize(600, 400);
        imageLabel->setStyleSheet("QLabel { background-color : lightgray; border: 1px solid black; }");
        imageLayout->addWidget(imageLabel);

        upperLayout->addLayout(imageLayout);

        mainLayout->addLayout(upperLayout);

        QHBoxLayout* bottomButtonLayout = new QHBoxLayout;
        bottomButtonLayout->addStretch();


        prevButton = new QPushButton("previous", this);
        prevButton->setFixedSize(100, 40);
        prevButton->setEnabled(false);
        bottomButtonLayout->addWidget(prevButton);
        connect(prevButton, &QPushButton::clicked, this, &SimpleAnnotationTool::previousImage);

        QPushButton* saveButton = new QPushButton("save label", this);
        saveButton->setFixedSize(150, 40);
        bottomButtonLayout->addWidget(saveButton);
        connect(saveButton, &QPushButton::clicked, this, &SimpleAnnotationTool::saveAnnotations);

        nextButton = new QPushButton("next", this);
        nextButton->setFixedSize(100, 40);
        nextButton->setEnabled(false);
        bottomButtonLayout->addWidget(nextButton);
        connect(nextButton, &QPushButton::clicked, this, &SimpleAnnotationTool::nextImage);
        bottomButtonLayout->addStretch();

        mainLayout->addLayout(bottomButtonLayout);

        QVBoxLayout* labelButtonLayout = new QVBoxLayout;
        labelButtonLayout->setSpacing(10);

        for (const QString& label : allLabels) {
            QPushButton* button = new QPushButton(label, this);
            button->setFixedSize(150, 40);
            button->setEnabled(false);
            connect(button, &QPushButton::clicked, this, [=]() { annotate(label); });
            labelButtonLayout->addWidget(button);
            annotationButtons.append(button);
        }

        upperLayout->addLayout(labelButtonLayout);
    }

    void enableAnnotationButtons(bool enable) {
        for (QPushButton* button : annotationButtons) {
            button->setEnabled(enable);
        }
    }
    bool annotationsModified=false;
    QLabel* imageLabel;
    QLabel* imageInfoLabel;
    QLabel* imageStatusLabel;
    QListWidget* annotationList;
    QPushButton* prevButton;
    QPushButton* nextButton;
    QStringList imageFiles;
    QString imageDir;
    QStringList annotations;
    QStringList allLabels = {"水位_VA","破裂_RB","表面损伤_OB","生产错误_PF","变形_DE","脱节FS","接口材料脱落IS","树根RO",
        "渗透IN","沉积AF","结垢BE","障碍物FO","支管暗接GR","凿孔连接PH","钻孔连接PB","侧向修复切口OS",
        "施工变更相关缺陷","过度刨面相关缺陷","无缺陷ND","有缺陷Defect"
    };
    QList<QPushButton*> annotationButtons;
    int currentIndex;
};

int main(int argc, char* argv[]) {
    QApplication app(argc, argv);
    SimpleAnnotationTool window;
    window.setWindowTitle("simple label tool");
    window.resize(800, 600);
    window.show();
    return app.exec();
}

#include "main.moc"
