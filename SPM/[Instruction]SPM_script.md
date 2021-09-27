# SPM_Batch_Generator.ipynb
   
   `Notice` These don't generate preprocessed image, but scripts for preprocessing.
   
   This notebook generates script of SPM preprocessing which includes `DICOM import`, `Realign & Slice time correction`, and `Coregister & Segment & Normalization & Smoothing`.
   
   It also includes `fMRI model specification`, `Model estimation`, `Contrast Manager`. 
   
   The parameters are default value of SPM. If you want, edit following parameters.

---
   - Main preset
  
  ``` python
  Sequence = { 'Anatomy': 'BRAVO'   ,   # Folder name containing anatomy images of each subject (ex) BRAVO
              'fMRI'   : 'NEGATIVE' }   # Folder name containing EPI images of each subject (ex) NEGATIVE
 Folder_Structure = '/*/*/'            # Folder structure of data. (ex) Subject / sequence
  ```
---
  - Realign & Slice timing
  
  ``` python
  realign = {
    ##Estimation Options##
    'Quality'      : 0.9,       #0~1, higher the better, lower the faster
    'Separation'   : 4,         #The separation (in mm) between the points sampled in the reference image
    'Smoothing'    : 5,         #The FWHM of the Gaussian smoothing kernel (mm) applied to the images before estimating
    'Num_Passes'   : 1,         #0: Register to first, 1: Register to mean
    'Interpolation': 2,         #0: Trilinear, 2~7: n-th Degree B-Spline
    'Wrap'         : '[0 0 0]', #[X, Y, Z], 0: no wrapping, 1: wrapping for the P.E direction
    ##Reslice Options##
    'Masking'      : 1,       #Searching through the whole time series looking for voxels which need to be 
                              #sampled from outside the original images.
    'fname_prefix' : 'r'      #default: 'r'
}

sliceTiming = {
    'Number of slices': 'AUTO',  #Number of slices will be filled automatically.
    'TR'              : 3     ,  #n * 1000ms
    'TA'              : 'AUTO',  #TA will be filled automatically.
    'Slice_order'     : 'AUTO',  #slice order will be filled automatically
    'Reference slice' : 1     ,  #If slice times are provided instead of slice indices in the previous item, this value should
                                 #represent a reference time (in ms) instead of the slice index of the reference slice.
    'fname_prefix' : 'a'         #default: 'a'
}
  ```  
---
  - Coregister & Segment & Normalize & Smoothing
 ``` python
 coregister = {
    'Objective_Function' : 'nmi',   #nmi: Normalized Mutual Information
    'Separation'         : '[4 2]', #The average distance between sampled points (in mm)
    'Tolerance'          : '[0.02 0.02 0.02 0.001 0.001 0.001 0.01 0.01 0.01 0.001 0.001 0.001]',      
                                    #Iterations stop when differences between successive estimates are less thean the required tolerance
    'Smoothing'          : '[7 7]'  #Gaussian smoothing to apply to the 256x256 joint histogram.
}

Segment = {
    'Bias_regularization'     : 0.001,   #0 / 0.00001 / 0.0001 / 0.01 / 0.1 / 1 / 10 ==> ex) If bias is very little, set lower value
    'Bias_FWHM'               : 60,      #mm, 30 / 40 / 50 / 60 / 70 / 80 / 90 / 100 / 110 / 120 / 130 / 140 / 150 / None
    'Save_corrected_and_field': '[0 1]', #[Field, corrected]
    'Num_gaussians'           : [1,1,2,3,4,2], #0~8, 
                                         #Used to represent the intensity distribution for each tissue class can be greater than one
    'Native_Tissue'           : [ '[1 1]', '[1 1]', '[1 1]', '[1 0]', '[1 0]', '[0 0]' ], #[Native, Dartel]
                                         #Produce a tissue class image (C*) that is in alignment with the original.[Dartel toolbox(rc*)]
    'Warped_Tissue'           : [ '[1 0]', '[1 0]', '[1 0]', '[1 0]', '[1 0]', '[1 0]' ], #[Modulated, Unmodulated]
                                         #Produce spatially normalised versions of the tissue class. (mwc* and wc*)
    'MRF_Filter'              : 1,       #Strength of MRF
                                         #Cleanup procedure for tissue class images with few iterations of a simple Markov Random Field 
    'Clean_up'                : 1,       #0: Don't cleanup, 1: Light Clean, 2: Thorough clean
    'Warping_regularization'  : '[0 0.001 0.5 0.05 0.2]', #Read SPM manual for detailed instruction
                                         #Measure of similarity between images and measure of the roughness of the deformation
    'Affine_Regularization'   : 'eastern', 
    #No Affine Registration: ''
    #No regularisation: 'none'
    #ICBM space template - East Asian Brains: 'eastern'
    #ICBM space template - European: 'mni'
    #Average sized template: 'subj'
    'Smoothness'              : 0,         #Derive a fudge factor to account for correlations between neighbouring voxels 
    'Sampling_distance'       : 3,         #Encodes the approximate distance between smapled points when estimating the model parameters
    'Deformation_Fields'      : '[0 1]' #[Inverse, Forward]
                                    #Forward: spatially normalizing image to MNI / Inverse: spatailly normalizing GIFTI surface files
}

Normalize = {
    'Bounding_box' : [-78, -112, -70, 78, 76, 85],
    'Voxel_sizes'  : '[2 2 2]',
    'Interpolation': 4,
    'fname_prefix' : 'w',
    }

Smoothing = {
    'FWHM'         : '[10 10 10]',
    'Data_type'    : 0        , #0: Same dtype as original, 1: UINT8, 2: INT16, 3: INT32, 4: Float32, 5: Float64
    'Implicit_mask': 0        , 
    'fname_prefix' : 'FINAL_' 
}
 ```
---
   - fMRI model specification
 ``` python
Fixed_path = False
SPM_MAT_list = []
Specification = {
    'SPM.mat_dir': 'D:/Workspace/TEST/Control/SPM_MAT' if Fixed_path else filedialog.askdirectory(title='Where to save SPM.mat files?', 
                                                                                            initialdir=os.getcwd()),
    'Units_for_design'    : 'secs', #'secs'
    'Interscan_interval'  : 3,  #TR
    'Microtime_resolution': 16,
    'Microtime_onset'     : 8,
    'Session_total'       : 3,
    'Session_info'        :[ 
            #Session name             (ex) If there are two sessions, [ 'Task', 'Fixation' ]
            ['A', 'B', 'Fixation'], 
            #Oneset times per session (ex) If there are two sessions, [ [0, 84, 120, 156], [36, 72, 132, 144, 216] ]  
            [  [0.75, 5.5,10.25,15,19.75,24.5,29.25,34,38.75,43.5,48.25,53,228.75,233.5,238.25,243,247.75,252.5,257.25,262,266.75,
                271.5,276.25,281,342.75,347.5,352.25,357,361.75,366.5,371.25,376,380.75,385.5,390.25,395],
               [57.75,62.5,67.25,72,76.75,81.5,86.25,91,95.75,100.5,105.25,110,171.75,176.5,181.25,186,190.75,195.5
               ,200.25,205,209.75,214.5,219.25,224,399.75,404.5,409.25,414,418.75,423.5,428.25,433,437.75,442.5,447.25,452],
               [114,285,456]            ],
            #Duration                 (ex) If there are two sessions, [ 10, 5 ] 
               [3.5, 3.5, 57.0], 

            #[], #Time modulations, 0: None, 1~6: order of time modulation (ex) If there are two sessions, [0, 0] ==> Working on it :<
            #[], #Parametric modulations   ==> Working on it :<
            #[], #Orthogonalise modulations, 0: no, 1: yes, (ex) If there are two sessions, [1, 1]] ==> Working on it :<
    ],
    #'Multiple_conditions': '', Working on it :<
    #'Regressors'
    'Multiple_regressor'  : 'rp', #Type prefix of multiple regressor file
    'High_Pass_filter'    : 128,
    #'Canionical_HRF'      : '[0 0]', Working on it :<
    #'Global_normalization': 'None', Working on it :<
    'Masking_threshold'   : 0.8,
    #'Explicit_mask'      : '', Working on it :<
    #'Serial_correlations : 'AR(1)', Working on it :<
}
```
---
   - Contrast Manager
      
      Sessions are defined at fMRI model specification.
      
      --> [session1, session2, session3...]
      
 ``` python
 Contrast = {
    'A-B'   : '[-1 1 0]',
    'A-Fix' : '[0 1 -1]',
    'B-Fix' : '[1 0 -1]',
}
```
