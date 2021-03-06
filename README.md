# Lecture 9 : Protyping at the scanner with MATLAB part 2

Title : Protyping at the scanner with MATLAB part 2

Schedule : June 25, 2020 | 16:00-17:00 

Speakers : Stanislas Rapacchi & Aurélien Trotier

[TOC]


## Foreword 

The Gadgetron responds to two major issues in MRI:
- prototyping: how to develop a new reconstruction and associate it with an existing or developing sequence.
- deployment: how to deploy a sequence and reconstruction on several sites for a clinical study

For many, Matlab remains the environment of choice for rapid prototyping in signal and image processing. This session will target such needs. 

Basic knowledge of Matlab is expected for this session. Advanced Matlab programming such as packaging and nested functions is not mandatory and can be easily understood on-the-fly. Gadgetron-Matlab uses both, and we hope this session will demystify these notions for those not familiar with them.

## Installation

To do the tutorial, you need to install three components:

* MATLAB **R2018b and above** (Superior to R2017a is also possible but need a modification of gadgetron) 
* [gadgetron](https://github.com/gadgetron/gadgetron)
* [gadgetron-matlab](https://github.com/gadgetron/gadgetron-matlab)



**Gadgetron**

Detailed installation instructions have been summarized [here](https://github.com/gadgetron/GadgetronOnlineClass/tree/master/Installation). But basically, on Ubuntu you need to run the following line:

```
sudo add-apt-repository ppa:gradient-software/experimental
sudo apt-get update
sudo apt-get install gadgetron-all
```



**Matlab installed in Windows and Gadgetron in WSL**

For a Windows Matlab installation to be visible from WSL, an extra script is needed. Create a script file named 'matlab' (no extension!). You can use nano or your preferred editor:

 `nano matlab`

In it, copy:

```
#!/bin/bash
export WSLENV=GADGETRON_EXTERNAL_PORT:GADGETRON_EXTERNAL_MODULE
matlab.exe $@
```

And save the file 'matlab'. Make it executable:

```
sudo chmod u+x matlab
```

Move it to a proper place that is in your PATH:

```
mv matlab /usr/local/bin/matlab
```

Save. Once installed, we're good to go!



**Matlab version between R2017a and R2018a**

You need to install gadgetron from the source code and edit the file **gadgetron/connection/stream/external/Matlab.cpp**

Replace line 16 :
`boost::process::args={"-batch", "gadgetron.external.main"},'`  
by  
`boost::process::args={"--nosplash", "--nodesktop", "-r",  "\"gadgetron.external.main; exit\""},`  
**Need to be tested**

Recompile gadgetron.



**Gadgetron-Matlab**

Detailed installation and how to install and use Gadgetron-Matlab is available [here](https://github.com/gadgetron/gadgetron/wiki/Using-Matlab-with-Gadgetron). Basically you can search **gadgetron** in the Matlab Add-On manager.



**Optional**

The following program will be used at the end of the tutorial. You can skip this part at the beginning.
[ismrmrdviewer](https://github.com/ismrmrd/ismrmrdviewer)
[BART](https://github.com/mrirecon/bart)



## Testing your installation

To verify the Matlab-Gadgetron connection is working, one can type 

`gadgetron --info.` 

Matlab should be supported.  

```
Gadgetron Version Info
  -- Version            : 4.1.1
  -- Git SHA1           : 74f1d293866bb46a7c75ccca3eb0ededb4911e72
  -- System Memory size : 32647 MB
  -- Python Support     : YES
  -- Matlab Support     : YES
  -- CUDA Support       : NO
```

More thoroughly, after installing Gadgetron it is always a good idea to run the integration test. Here we will run the Matlab test.

To do so move to the folder **gadgetron/test/integration**/

Download all the datasets with :

`python get_data.py`

and then run the matlab test with :

```python run_tests.py cases/external_matlab_tiny_example.cfg```

If it is working you will see an output with this indication at the end : 

```
Test status: Passed

1 tests passed. 0 tests failed. 0 tests skipped.
```



## Sequence and Data

We will use a 3D MP2RAGE sequence with a variable density poisson undersampling mask acquired on a 3T Prisma from Siemens.

Data is available at this link [![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.3777994.svg)](https://doi.org/10.5281/zenodo.3777994): **Need to be change**

One dataset (without noise calibration) is available: 

- brain, 0.8mm isotropic, acceleration factor = 20. 

The data has been converted with **siemens_to_ismrmrd**, we will not discuss data conversion here. This will be the object of the following readings.



## Objectives

* Review formalism of Matlab scripts and code
  * Classes, packages in +folder, nested functions
  * Data types definition
  * Retrieve protocol headers (former xml) in Matlab
  * Connection next actions
* Understand how data is stored under matlab depending of your xml configuration file :
  * Before AcquisitionAccumulateTriggerGadget
  * After AcquisitionAccumulateTriggerGadget
  * After BucketToBufferGadget
* Create a new matlab gadget using :
  * the matlab debugger : **<connect>**
  * launching it without a matlab session open : <execute>
* Send back images
* Call BART for advanced reconstruction method



# Session layout

### Review formalism of Matlab scripts and code

* Classes, packages in +folder, nested functions

Matlab can package a set of functions and classes in a folder starting with a `+` sign. By adding the parent folder in Matlab PATH, the whole package is exposed and can be called subsequently. If the gadgetron add-on has been added, from the `+gadgetron` folder, the class Constants can be called in Matlab:

```
>>gadgetron.Constants.ERROR

ans =

  uint16

   8
```

A lot of nested functions are employed in the following demonstration. Nested functions have a limited scope of definition.

```matlab
function next = noise_adjust(input, header)

noise_matrix        = [];

function transformation = calculate_whitening_transformation(data)
    covariance = (1.0 / (size(data, 1) - 1)) * (data' * data); 
    transformation = inv(chol(covariance, 'upper'));
end

function acquisition = apply_whitening_transformation(acquisition)
    acquisition.data = acquisition.data * noise_matrix; 
end

function acquisition = handle_noise(acquisition)    
        if acquisition.is_flag_set(acquisition.ACQ_IS_NOISE_MEASUREMENT)
   noise_matrix = calculate_whitening_transformation(acquisition.data);
        else 
   acquisition = apply_whitening_transformation(acquisition);
        end
end

next = @() handle_noise(input());

end
```

The variable noise_matrix is shared among all nested functions. One can define it, while another can use it internally. Safeguards have been remove for clarity, they are the user's responsibility! 

* Data types definition
* Retrieve protocol headers (former xml) in Matlab
* Connection next actions

## Objectives

- to become familiar with the Cartesian reconstruction pipeline
- to create new python gadget from scratch
- to create a new xml configuration file 
- data manipulation (readout, kspace, image)
- to call BART from a python gagdet
- to call SigPy from a python gagdet

## A typical Python Gadget


```python
import numpy as np
import gadgetron
import ismrmrd

def EmptyPythonGadget(connection):
   
   for acquisition in connection:
          
       # DO SOMETHING     
       connection.send(acquisition)
  

```
The function is responsible for receiving all messages from the previous gadget and for sending a new message to the next gadget using **connection.send()**
It may or may not interact with the information contained in the message.  

## My first Python Gadget

Create a new directory named `GT_Lecture3` and open two terminals at the location.

```
mkdir GT_Lecture3
cd GT_Lecture3
```

### Writing the gadget

Create the file my_first_python_gadget.py then copy the previous class. 

```
We can add the following message before connection.send() that we are going through it.
print("so far, so good")
```


### Writing the XML chain 

We will now create a new xml file named `external_python_tutorial.xml`. Add the following content into `external_python_tutorial.xml`

```

<?xml version="1.0" encoding="utf-8"?>
<gadgetronStreamConfiguration xsi:schemaLocation="http://gadgetron.sf.net/gadgetron gadgetron.xsd"
        xmlns="http://gadgetron.sf.net/gadgetron"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">

    <!--
        Gadgetron generic recon chain for 2D and 3D cartesian sampling
        
    -->

    <!-- reader -->
    <reader><slot>1008</slot><dll>gadgetron_mricore</dll><classname>GadgetIsmrmrdAcquisitionMessageReader</classname></reader>
    <reader><slot>1026</slot><dll>gadgetron_mricore</dll><classname>GadgetIsmrmrdWaveformMessageReader</classname></reader>

    <!-- writer -->
    <writer><slot>1022</slot><dll>gadgetron_mricore</dll><classname>MRIImageWriter</classname></writer>

    


</gadgetronStreamConfiguration>
```

To call our gadget, we have to add it to the reconstruction chain which is currently empty. For this add the following lines after the `` MRIImageWriter``

```
<stream>  
        <external>
            <execute name="my_first_python_gadget" target="EmptyPythonGadget" type="python"/>
            <configuration/>
        </external> 
</stream>
```

## Compilation and installation 

Nothing to do.

### Reconstruction and visualisation

To run the reconstruction chain, you'll need to run Gadgetron, and the Gadgetron ISMRMRD client. 

Start Gadgetron:
```bash
$ gadgetron
```

Run the ISMRMRD client: 
```bash 
$ cd path/to/gadgetron/test/integration
$ gadgetron_ismrmrd_client -f in.h5  -C external_python_tutorial.xml
```

```
gadgetron_ismrmrd_client -f     -C python_passthrough_tutorial.xml

```

Les données sont écrites par défaut dans un fichier out.h5.

Utiliser le script python pour les lire et afficher les images

```

```


### Exercice 1 : find the number of readout

Ajouter les lignes suivantes dans la fonction process.

```
self.counter=self.counter+1;
print(self.counter)
```

Puis rejouer l'étape compilation et installation et lancer la reco.

### Exercice 2 : display the matrix size

Ajouter les lignes suivantes dans la fonction process et import numpy as np en entête

```
print(np.shape(data))
```

Puis rejouer l'étape compilation et installation et lancer la reco.

### Exercice 3 : display the AcquisitionHeader

Ajouter les lignes suivantes

print(header)


Nous allons obtenir ce type de message

```
2404
(256, 12)
version: 1
flags: 2097152
measurement_uid: 26
scan_counter: 2404
acquisition_time_stamp: 22684240
physiology_time_stamp: 8545183, 0, 0
number_of_samples: 256
available_channels: 12
active_channels: 12
channel_mask: 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0
discard_pre: 0
discard_post: 0
center_sample: 128
encoding_space_ref: 0
trajectory_dimensions: 0
sample_time_us: 1.600000023841858
position: 0.0, -4.842615127563477, -75.69005584716797
read_dir: -6.123031769111886e-17, 1.0, 0.0
phase_dir: 1.0, 6.123031769111886e-17, 0.0
slice_dir: 0.0, 0.0, 1.0
patient_table_position: 0.0, 0.0, -1259016.0
idx: kspace_encode_step_1: 23
kspace_encode_step_2: 0
average: 0
slice: 0
contrast: 0
phase: 0
repetition: 2
set: 0
segment: 1
user: 0, 0, 0, 0, 0, 32, 0, 0

user_int: 0, 0, 0, 0, 0, 0, 0, 0
user_float: 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0
```

### Exercice 4 : readout selection and corresponding encoding space

Récupérer le numero de repetition et de slice avec les lignes suivantes:

```
slice = header.idx.slice
repetition=  header.idx.repetition
e1=header.idx.kspace_encode_step_1
segment=header.idx.segment
```

Et ajouter la condition suivante pour ne pas transmettre tous les readouts des autres coupes et autres répétitions.

```
if (repetition>0 and slice>0):
   print(self.counter_send, " slice: ",slice , " rep: ", repetition, " e1: ", e1," segment: ",  segment)
    
```

Ajouter un second compteur nommé self.counter_send qui s'incrémente dans cette boucle.



### Exercice 5 : Buffering

self.myBuffer = None

self.enc = None

Dans process config ajouter ceci pour récupérer la taille de la amtrice:

self.header = ismrmrd.xsd.CreateFromDocument(conf)
self.enc = self.header.encoding[0]


```
if self.myBuffer is None:
     channels = acq.active_channels
            
     eNz = self.enc.encodedSpace.matrixSize.z
     eNy = self.enc.encodedSpace.matrixSize.y
     eNx = self.enc.encodedSpace.matrixSize.x
        
     self.myBuffer = np.zeros(( int(eNx),eNy,channels),dtype=np.complex64)

self.myBuffer[:,e1,:] = data
```

### Exercice 6 : Fourier Transform

```
from matplotlib import transform
from ismrmrdtools import transform


if (e1==96)
    plt.figure(1)    
    plt.imshow(np.abs(self.myBuffer[:,:,0]))
```

### First Conclusion

Pas forcément nécessaire de tout redévelopper

Il existe de nombreux gadgets en python et/ou en C++ qui permettent d'aller plus vite.


TODO: installation sans GPU / avec GPU
      docker avec/sans Python / Matlab












## A brief description of the class used to store readout, kspace or image data

The data structures in the gadgetron vary during reconstruction. It is important to differenciate, the class or common structures 

* used to store a unit of readout that would feed into a buffer
* used to store a unit of data that would feed into a reconstruction
* used to store an array of reconstructed data

Each of them are defined in a C++ and have equivalent in Python. Additionnal structure are also present and you can create new one

### Readout

Le gadget python recevra deux messages associés qui contiennent le **AcquisitionHeader** et les données sous forme de matrice **hoNDArray< std::complex<float> >**.
En python le **hoNDArray** est la matrice multidimensionnel **ndarray** issue de la librairie numpy 

``` 
process(self, header, data):

```

```
print(type(header))

print(type(data))
```

### Kspace

En imagerie cartesienne, deux gadgets jouent un role fondamental : AcquisitionAccumulateTriggerGadget et BucketToBufferGadget. 

Ces gadgets servent à bufferiser les readouts afin de construire le kspace. En IRM, les dimensionalités sont très nombreuses: 

* kx (RO)
* ky (E1)
* kz (E2)
* channels (CHA)
* average
* repetition
* segment
* contrast
* phase
* set
* slice (SLC)
* ...

Par convention, nous aurons en entrée des matrices de dimentions [RO, CHA] et en sortie de BucketToBufferGadget des matrices de dimensions [RO E1 E2 CHA N S SLC].
Les dimensions **N** et **S** peuvent être choisie arbitrairement par l'utilisateur. 

Il est fort interessant de se positionner après ces gadgets ou sont automatiquement triés les données kspaces, que ce soit les lignes de calibration en imagerie parallèle ou les lignes sous échantionnées.  


Les données de calibration si elles sont présentes sont accessibles via la structure suivante:

```
buffer.ref.data  
buffer.ref.header
```

Les données "standard" sont accessibles via la structure suivante:

```
buffer.data.data
buffer.data.header
```

Attention, la taille des headers est associée la taille des données, les headers sont généralement différents, par ex la position des coupes changent suivant la direction SLC.
Nous avons donc maintenant une matrice hondarray de acquisitionheader de dimensions [E1 E2 N S SLC]. Les headers étant identique suivant la direction de readout et pour tous les éléments d'antennes.

 


Le gadget python correspondant recevra un message qui contient une structure nommé **IsmrmrdReconBit** nommée IsmrmrdReconBit et IsmrmrdDataBuffered. Le gadget python doit donc contenir les includes suivants.   

``` 
from gadgetron import Gadget,IsmrmrdDataBuffered, IsmrmrdReconBit, SamplingLimit,SamplingDescription, IsmrmrdImageArray


process(self, buffer):

```

Pour aller plus loin, voici les classes C++ correspondantes.



```
struct IsmrmrdReconBit
  {
  public:
    IsmrmrdDataBuffered data_;
    boost::optional<IsmrmrdDataBuffered> ref_;
  }

```

```
struct IsmrmrdDataBuffered
  {
  public:
    //7D, fixed order [E0, E1, E2, CHA, N, S, LOC]
    hoNDArray< std::complex<float> > data_;
    
    //7D, fixed order [TRAJ, E0, E1, E2, N, S, LOC]
    boost::optional<hoNDArray<float>> trajectory_;

    // 6D, density weights [E0, E1, E2, N, S, LOC]
    boost::optional<hoNDArray<float> > density_;

    //5D, fixed order [E1, E2, N, S, LOC]
    hoNDArray< ISMRMRD::AcquisitionHeader > headers_;

    SamplingDescription sampling_;
   }

```


### Image


### Writing the Buffered Gadget

Nous allons donc maintenant créer un nouveau gadget nommé MyFirstDataBufferedGadget.



```python
import sys
import ismrmrd
import ismrmrd.xsd

from gadgetron import Gadget,IsmrmrdDataBuffered, IsmrmrdReconBit, SamplingLimit,SamplingDescription, IsmrmrdImageArray

class MyFirstDataBufferedGadget(Gadget):
    def __init__(self,next_gadget=None):
        super(MyFirstDataBufferedGadget,self).__init__(next_gadget=next_gadget)
        self.my_value = 0
        self.my_list = []
	self.my_matrix =[]

    def process_config(self, conf):
        # do allocation        
	pass 

    def process(self, message):
               
	# get information from the message

	# modify the message

	# send the message to the next gadget
        self.put_next(message)
        return 0

```


### Compilation, Installation and launching the reco
### Exercice 1: Fourier Transform using imsmrmrd-python-tool
### [Exercice 2: Fourier Transform using BART
### Exercice 3: Fourier Transform using Sigpy
### Exercice 4: Grappa reconstruction using PyGrappa