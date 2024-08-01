# CourtCheck <img src="https://github.com/AggieSportsAnalytics/CourtCheck/blob/cory/images/courtcheck_ball_logo.png" alt="CourtCheck Logo" style="width: 80px; vertical-align: middle;"> 

### 🏁 Automate tennis match analysis using the power of computer vision.

CourtCheck leverages advanced computer vision techniques to accurately track tennis ball movements and court boundaries in tennis matches. This project aims to provide real-time insights and automated decision-making in tennis, reducing human error and enhancing the accuracy of in/out calls. The project integrates Python, machine learning, and computer vision to create a seamless and efficient system for tennis match analysis.

![courtcheck-demo](https://github.com/AggieSportsAnalytics/CourtCheck/blob/main/images/game2_processed_10s.gif)

# 🔑 Key Features



## 🔎 Court Detection

CourtCheck employs keypoint detection algorithms to identify and track the tennis court's boundaries, ensuring accurate mapping and analysis of the court's dimensions.

### 📑 Annotation

We started by annotating images using OpenCV in the COCO format, generating JSON files for each annotated image. The OpenCV Annotation Tool has a fantastic interface to annotate images and export them in different formats. It also features a great interpolation tool that allows the use of a skeleton, enabling the labeling of key frames that can be interpolated and applied over consecutive frames in the video.

![annotation-demo](https://github.com/AggieSportsAnalytics/CourtCheck/blob/cory/images/opencv_annotation.gif)

You can find the annotations [here](https://drive.google.com/drive/folders/16HugQeDoXUX420yKjg7pGVu3NG6linMV?usp=drive_link).

Each label in the skeleton represents a keypoint on the tennis court, identifying an important corner or intersection of lines that are crucial for the overall court detection when training the model. Here are the keypoints and their corresponding labels:

| Keypoint | Label                       | Keypoint | Label                           | Keypoint | Label                       |
|----------|-----------------------------|----------|---------------------------------|----------|-----------------------------|
| BTL      | Bottom Top Left             | ITM      | Inner Top Middle                | ITR      | Inner Top Right             |
| BTLI     | Bottom Top Left Inner       | IBR      | Inner Bottom Right              | NL       | Net Left                    |
| BTRI     | Bottom Top Right Inner      | NR       | Net Right                       | BBL      | Bottom Bottom Left          |
| BTR      | Bottom Top Right            | NM       | Net Middle                      | IBL      | Inner Bottom Left           |
| BBR      | Bottom Bottom Right         | ITL      | Inner Top Left                  | IBM      | Inner Bottom Middle         |
| BBRI     | Bottom Bottom Right Inner   |          |                                 | BBLI     | Bottom Bottom Left Inner    |

### 🤖 Training the Model

We utilized the A100 Nvidia GPU to train our Detectron2 model on different types of datasets. These datasets included varying court surfaces and slightly different camera angles to ensure robustness and generalizability of the model. Below, we explain the process and provide the code used for training the model incrementally with mixed datasets.

Below is an overview of the Detectron2 architecture:

<div align="center">
    <img src="https://github.com/AggieSportsAnalytics/CourtCheck/blob/cory/images/detectron2_architecture.png" alt="Detectron2 Architecture" width="700"/>
</div>
<br>

We used the `COCO-Keypoints/keypoint_rcnn_R_50_FPN_3x.yaml` configuration file because it is specifically designed for keypoint detection tasks. The [keypoint_rcnn_R_50_FPN_3x.yaml](https://drive.google.com/drive/folders/18t8oUo5_jzYYD1vnFzjLw7uhxmagXINX?usp=drive_link) configuration is well-suited for this task because it includes a pre-trained ResNet-50 backbone that provides strong feature extraction capabilities, coupled with a Feature Pyramid Network (FPN) that helps detect objects at multiple scales. This combination ensures that the model can accurately identify and track the key points on the tennis court, providing precise court boundary detection and enabling accurate in/out call determinations.


#### 🧬 Model Code

The code below sets up and trains the Detectron2 model using multiple datasets:

1. **Dataset Registration**: Registers the training and validation datasets.
2. **COCO Instance Registration**: Registers the datasets in COCO format.
3. **Metadata Configuration**: Configures metadata for keypoints, keypoint flip map, and skeleton.
4. **Configuration Setup**: Sets up the model configuration, including dataset paths, data loader workers, batch size, learning rate, maximum iterations, learning rate decay steps, and checkpoint period.
5. **Trainer Initialization and Training**: Initializes a custom trainer and starts or resumes the training process.

You can find the Google Colab Notebook [here](https://colab.research.google.com/drive/1huJ4f0yOApwM4NR8gpXIHktHkTgrbL_m?usp=drive_link).

```python
# Function to set up and train the model with mixed datasets incrementally
def train_model(max_iter, resume=False):
    register_datasets(train_json_files, train_image_dirs, "tennis_game_train")
    register_datasets(val_json_files, val_image_dirs, "tennis_game_val")

    register_coco_instances(f"tennis_game_train", {}, train_json_files, train_image_dirs)
    register_coco_instances(f"tennis_game_val", {}, val_json_files, val_image_dirs)

    MetadataCatalog.get(f"tennis_game_train").keypoint_names = keypoint_names
    MetadataCatalog.get(f"tennis_game_train").keypoint_flip_map = keypoint_flip_map
    MetadataCatalog.get(f"tennis_game_train").keypoint_connection_rules = skeleton

    MetadataCatalog.get(f"tennis_game_val").keypoint_names = keypoint_names
    MetadataCatalog.get(f"tennis_game_val").keypoint_flip_map = keypoint_flip_map
    MetadataCatalog.get(f"tennis_game_val").keypoint_connection_rules = skeleton

    cfg = get_cfg()
    cfg.merge_from_file("/content/drive/MyDrive/ASA Tennis Bounds Project/models/court_detection_model/detectron2/configs/COCO-Keypoints/keypoint_rcnn_R_50_FPN_3x.yaml")
    cfg.DATASETS.TRAIN = tuple([os.path.basename(f).split('.')[0] for f in train_json_files])
    cfg.DATASETS.TEST = tuple([os.path.basename(f).split('.')[0] for f in val_json_files])
    cfg.DATALOADER.NUM_WORKERS = 4
    cfg.SOLVER.IMS_PER_BATCH = 4  # Increase if you have more GPU memory
    cfg.SOLVER.BASE_LR = 0.0001  # Lower learning rate for more careful training
    cfg.SOLVER.MAX_ITER = max_iter  # Total number of iterations
    cfg.SOLVER.STEPS = [int(max_iter*0.75), int(max_iter*0.875)]  # Decay learning rate
    cfg.SOLVER.GAMMA = 0.1  # Decay factor
    cfg.SOLVER.CHECKPOINT_PERIOD = 20000  # Save a checkpoint every 20000 iterations
    cfg.MODEL.ROI_HEADS.BATCH_SIZE_PER_IMAGE = 256  # Increase for more stable gradients
    cfg.MODEL.ROI_HEADS.NUM_CLASSES = 11  # Your dataset has 11 classes

    output_dir = f"/content/drive/MyDrive/ASA Tennis Bounds Project/models/court_detection_model/detectron2/game_model"
    cfg.OUTPUT_DIR = output_dir
    os.makedirs(cfg.OUTPUT_DIR, exist_ok=True)

    trainer = TrainerWithEval(cfg)
    trainer.resume_or_load(resume=resume)
    trainer.train()

# Training parameters
custom_iter = 50000  # Adjust this to your custom number of iterations per session
max_iter = last_iter + custom_iter  # Change this for the number of iterations per session

# Execute to train model
train_model(max_iter, resume=True)
```

### 📽️ Post Processing

After training the model, the next crucial step is post-processing the results to ensure accurate and meaningful outputs. Post-processing involves refining the model's predictions and visualizing the detected key points on the tennis court for better interpretation and analysis.

To visualize the court, we draw polylines between key points that align with the court lines and boundaries. These lines help in creating a clear and precise representation of the tennis court structure. Here are the specific polylines to be drawn:

```python
lines = [
    ("BTL", "BTLI"), ("BTLI", "BTRI"), ("BTL", "NL"), ("BTLI", "ITL"),
    ("BTRI", "BTR"), ("BTR", "NR"), ("BTRI", "ITR"), ("ITL", "ITM"), ("ITM", "ITR"),
    ("ITL", "IBL"), ("ITM", "NM"), ("ITR", "IBR"), ("NL", "NM"), ("NL", "BBL"),
    ("NM", "IBM"), ("NR", "BBR"), ("NM", "NR"), ("IBL", "IBM"),
    ("IBM", "IBR"), ("IBL", "BBLI"), ("IBR", "BBRI"), ("BBR", "BBRI"),
    ("BBRI", "BBLI"), ("BBL", "BBLI"),
]
```

The `visualize_predictions` function is essential for visualizing model predictions on an input image. Here are two key parts of the function:

```python
outputs = predictor(img)
v = Visualizer(
    img[:, :, ::-1],
    metadata=MetadataCatalog.get("tennis_game_train"),
    scale=0.8,
    instance_mode=ColorMode.IMAGE,
)
instances = outputs["instances"].to("cpu")

if len(instances) > 0:
    max_conf_idx = instances.scores.argmax()
    instances = instances[max_conf_idx : max_conf_idx + 1]

out = v.draw_instance_predictions(instances)
keypoints = instances.pred_keypoints.numpy()[0]
```
This part of the function generates predictions from the model and selects the instance with the highest confidence score. The keypoints of this instance are extracted for further processing.

To ensure that the detected key points on the tennis court are stable and less jittery, especially when dealing with video frames, we use a stabilization technique. This involves averaging the positions of detected key points over a history of frames.

Key Point History Initialization
```python
keypoint_history = {name: deque(maxlen=10) for name in keypoint_names}
```
We initialize a dictionary called keypoint_history where each key is a key point name, and the value is a deque (double-ended queue) with a maximum length of 10. This deque will store the positions of each key point over the last 10 frames.

Stabalizing Keypoints
```python
def stabilize_points(keypoints):
    stabilized_points = []
    for i, keypoint in enumerate(keypoints):
        keypoint_history[keypoint_names[i]].append(keypoint[:2])
        if len(keypoint_history[keypoint_names[i]]) > 1:
            stabilized_points.append(
                np.mean(np.array(keypoint_history[keypoint_names[i]]), axis=0)
            )
        else:
            stabilized_points.append(keypoint[:2])
    return np.array(stabilized_points)
```
The `stabilize_points` function then uses the `keypoint_history` dictionary to process the detected key points and reduce jitter by averaging their positions over the last 10 frames. For each detected key point, its position is appended to the corresponding deque in the `keypoint_history` dictionary. If the deque contains more than one position, the average of these positions is computed and added to the `stabilized_points` list. If the deque contains only one position, the key point is added to the list as is. This results in more consistent and smooth key point positions for further processing and visualization.

#### Visualizing and Transforming the Court

To visualize the tennis court and transform it into a 2D plane, the following steps are taken:

1. **Court Detection in the Main Frame**:
    - The model detects key points on the tennis court in the main video frame.
    - Polylines are drawn between these key points to visualize the court lines and boundaries clearly, as shown in the first image.

2. **Transformation to Black and White and 2D Plane**:
    - The detected court is then converted into a black-and-white image to simplify the structure.
    - This black-and-white image is transposed into a 2D plane, providing a clear and concise representation of the tennis court, as depicted in the second image.

| Court Detection in Main Frame | Transposed 2D Plane |
|:-----------------------------:|:-------------------:|
| ![Court Detection in Main Frame](https://github.com/AggieSportsAnalytics/CourtCheck/blob/main/images/game1_court_processed.gif) | <img src="https://github.com/AggieSportsAnalytics/CourtCheck/blob/main/images/game2_2Dskeleton_10s.gif" alt="Transposed 2D Plane" style="width: 50%;"> |

Transforming Key Points

The `transform_points` function is designed to transpose the detected key points to fit within a defined black frame. This transformation ensures that the court is displayed correctly in a 2D plane. Below is an explanation of the function, broken down into its important parts:

Function Explanation
```python
def transform_points(keypoints, black_frame_width, black_frame_height):
    """Transform keypoints to fit within a black frame
    :params
        keypoints: list of keypoints to transform
        black_frame_width: width of the black frame
        black_frame_height: height of the black frame
    :return
        transformed_keypoints: transformed keypoints
        matrix: perspective transformation matrix
    """
    width_frac = 6
    height_frac = 7
```
The function takes in the key points, the width, and the height of the black frame. It uses fractions to determine the relative positions within the frame.

```python
    src_points = np.array(
        [
            keypoint_dict["BTL"],
            keypoint_dict["BTR"],
            keypoint_dict["BBL"],
            keypoint_dict["BBR"],
        ],
        dtype=np.float32,
    )
```
The destination points are defined within the black frame for the four corners of the court: BTL (Bottom-Top-Left), BTR (Bottom-Top-Right), BBL (Bottom-Bottom-Left), and BBR (Bottom-Bottom-Right). These points are used to establish the skeleton of the court within the black frame.

```python
    matrix = cv2.getPerspectiveTransform(src_points, dst_points)
    transformed_keypoints = cv2.perspectiveTransform(keypoints[None, :, :2], matrix)[0]
    return transformed_keypoints, matrix
```
Using OpenCV's `getPerspectiveTransform`, a transformation matrix is calculated to map the source points to the destination points. The perspective transformation is then applied to the key points, fitting them into the black frame for a top-down 2D view.


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
