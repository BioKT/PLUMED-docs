## Setting up PLUMED with Gromacs
First of all you must patch the Gromacs code with PLUMED. There
are multiple options available for you. 

### Installing with conda
If you only want
a hassle-free installation to try things out, then you can
simply use the pre-compiled versions made available by the 
PLUMED developers for their masterclasses. Here I follow
the instructions from the 
[first masterclass](https://www.plumed.org/doc-v2.7/user-doc/html/masterclass-21-1.html#masterclass-21-1-install)
In the terminal, type

	$ conda create --name plumed-masterclass

and then

	$ conda activate plumed-masterclass
	$ conda install -c conda-forge plumed py-plumed numpy pandas matplotlib notebook mdtraj mdanalysis git
	$ conda install --strict-channel-priority -c plumed/label/masterclass -c conda-forge gromacs

### Patch Gromacs with PLUMED 
First install PLUMED. In order to do this you will have to
download and unzip the software

	$ wget https://github.com/plumed/plumed2/archive/refs/heads/master.zip
	$ unzip master.zip
	$ cd plumed2-master/

Then you will follow the usual steps to install software using Make

	$ ./configure --prefix=/usr/local
	$ make
	$ sudo make install

You will also have to make sure that the installed libraries are 
visible in your path

	export LD_LIBRARY_PATH=/lib:/usr/lib:/usr/local/lib

The next step is to install Gromacs. First download the code
and unzip the package. In this case I am patching the 2021.6 
version of Gromacs.

	wget ftp://ftp.gromacs.org/gromacs/gromacs-2021.6.tar.gz
	tar -xvf gromacs-2021.6.tar.gz

The next thing I did was to check whether I could use Cmake to 
install with my desired options, in this case running with GPUs
and compiling with MPI support

	cd gromacs-2021.6/
	mkdir build
	cd build/
	cmake .. -DGMX_BUILD_OWN_FFTW=ON -DREGRESSIONTEST_DOWNLOAD=ON -DGMX_MPI=on -DGMX_GPU=CUDA -DCMAKE_INSTALL_PREFIX=/opt/gromacs/2021.6
make

Then comes the key step of running the ```plumed patch``` command

	cd ..
	plumed patch -p

After that you can go back to your Gromacs install directory and
finish the installation as usual

	cd build/
	cmake .. -DGMX_BUILD_OWN_FFTW=ON -DREGRESSIONTEST_DOWNLOAD=ON -DGMX_MPI=on -DGMX_GPU=CUDA -DCMAKE_INSTALL_PREFIX=/opt/gromacs/2021.6
	make
	sudo make install
	/opt/gromacs/2021.6/bin/gmx_mpi mdrun -h
