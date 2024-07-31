# CourtCheck <img src="https://github.com/AggieSportsAnalytics/CourtCheck/blob/cory/images/courtcheck_ball_logo.png" alt="CourtCheck Logo" style="width: 80px; vertical-align: middle;"> 

### 🏁 Automate tennis match analysis using the power of computer vision.

The CourtCheck Tennis Project leverages advanced computer vision techniques to accurately track tennis ball movements and court boundaries in tennis matches. This project aims to provide real-time insights and automated decision-making in tennis, reducing human error and enhancing the accuracy of in/out calls. The project integrates Python, machine learning, and computer vision to create a seamless and efficient system for tennis match analysis.

![courtcheck-demo](https://github.com/AggieSportsAnalytics/CourtCheck/blob/main/images/game2_processed_10s.gif)

# 🔑 Key Features



## 🔎 Court Detection

The project employs keypoint detection algorithms to identify and track the tennis court's boundaries, ensuring accurate mapping and analysis of the court's dimensions.

- ### 📑 Annotation

#### We started by annotating images using OpenCV in the COCO format, generating JSON files for each annotated image. This process involved marking key points on the tennis court to define the boundaries and important lines. These annotations are stored in JSON files.

[Link to JSON files and dataset](#)

- ### 🤖 Training the Model

#### Leveraging the GPUs available in Google Colab, we trained the Detectron2 model. The model was configured to detect and classify the key points of the tennis court from the annotated images.

Below is a summary of the code used for training the model in Google Colab:

```python
# Register the dataset
for d in ["train", "val"]:
    DatasetCatalog.register("tennis_" + d, lambda d=d: get_tennis_dicts("path/to/your/" + d))
    MetadataCatalog.get("tennis_" + d).set(thing_classes=["court"])

# Configure the model
cfg = get_cfg()
cfg.merge_from_file(model_zoo.get_config_file("COCO-Keypoints/keypoint_rcnn_R_50_FPN_3x.yaml"))
cfg.DATASETS.TRAIN = ("tennis_train",)
cfg.DATASETS.TEST = ()
cfg.DATALOADER.NUM_WORKERS = 2
cfg.MODEL.WEIGHTS = model_zoo.get_checkpoint_url("COCO-Keypoints/keypoint_rcnn_R_50_FPN_3x.yaml")
cfg.SOLVER.IMS_PER_BATCH = 2
cfg.SOLVER.BASE_LR = 0.00025
cfg.SOLVER.MAX_ITER = 1000
cfg.MODEL.ROI_HEADS.BATCH_SIZE_PER_IMAGE = 128
cfg.MODEL.ROI_HEADS.NUM_CLASSES = 1  # only has one class (court)
cfg.MODEL.KEYPOINT_ON = True

# Train the model
os.makedirs(cfg.OUTPUT_DIR, exist_ok=True)
trainer = DefaultTrainer(cfg)
trainer.resume_or_load(resume=False)
trainer.train()
```

## Ball Tracking

The system uses a deep learning model to track the tennis ball's position throughout the match, allowing for precise in/out call determinations.

- Utilizes a custom-trained TrackNet model for ball detection and tracking.

![ball-tracking-demo](https://github.com/SACUCD/CourtCheckTennis/assets/your-ball-tracking-image-url)
**_The yellow circle represents the detected position of the tennis ball._**

## 2D Court Simulation

One of the critical aspects of the project is transforming the detected ball and court positions onto a 2D map, enabling a clear and concise view of the ball's trajectory and court boundaries.

- Implements perspective transform using OpenCV.
- Transforms detected keypoints and ball positions to a 2D representation for easy analysis.

![2d-simulation-demo](https://github.com/SACUCD/CourtCheckTennis/assets/your-2d-simulation-image-url)
**_Red dots indicate transformed keypoints on the 2D court simulation._**

# 🪴 Areas of Improvement

- **Accuracy**: Enhance the accuracy of ball and court detection to ensure reliable analysis.
- **Real-Time Processing**: Implement real-time video feed analysis for instant decision-making during matches.
- **Automatic Court Detection**: Automate the court detection process to handle different court types and angles without manual input.
- **Player Tracking**: Integrate player tracking to provide comprehensive match statistics and insights.

# 🚀 Further Uses

- **Match Analytics**: Utilize the system for detailed match analytics, including player performance and shot accuracy.
- **Training and Coaching**: Provide coaches and players with valuable data to improve training sessions and match strategies.
- **Broadcast Enhancement**: Enhance sports broadcasting with real-time analysis and insights for viewers.

# 💻 Technology

- **OpenCV**: For image and video processing.
- **Detectron2**: For keypoint detection and court boundary identification.
- **TrackNet**: For tennis ball detection and tracking.
- **NumPy**: For numerical computations and data manipulation.
- **PyTorch**: For building and training machine learning models.
- **tqdm**: For progress bar visualization in loops and processing tasks.

## Installation

To set up the project, clone the repository and install the dependencies using the `requirements.txt` file:

```bash
git clone https://github.com/SACUCD/CourtCheckTennis.git
cd CourtCheckTennis
pip install -r requirements.txt
```
