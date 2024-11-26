# Delft3D installation on LENGAU runnning CentOS-7


Delft3D is Open Source Software and facilitates the hydrodynamic (Delft3D-FLOW module), morphodynamic (Delft3D-MOR module), waves (Delft3D-WAVE module), water quality (Delft3D-WAQ module including the DELWAQ kernel) and particle (Delft3D-PART module) modelling. The source code can be downloaded here: https://oss.deltares.nl/web/delft3d.

2018 Intel MPI compilation on CentOS-7

Using Tmux inside an interactive session,

**The set script:** Run the bash file that sets the delft3d environment. The file needs to be made executable and sourced to apply the changes. 
```
#!/bin/bash

module purge
module load chpc/parallel_studio_xe/18.0.2/2018.2.046
source /apps/compilers/intel/parallel_studio_xe_2018_update2/compilers_and_libraries/linux/mpi/bin64/mpivars.sh
module load chpc/BIOMODULES
module add curl/7.50.0
module add 

export DelftDIR=/home/apps/chpc/earth/delft3d
export DIR=$DelftDIR/LIBRARIES

export I_MPI_SHM="off"

export CC=icc
export CXX=icc
export FC=ifort
export FCFLAGS="-m64 -I$DIR/netcdf/include -I$DIR/grib2/include -I$DIR/hdf5-1.10.6/include"
export F77=ifort
export FFLAGS=$FCFLAGS
export CFLAGS=$FCFLAGS

export MPICC=/apps/compilers/intel/parallel_studio_xe_2018_update2/compilers_and_libraries/linux/mpi/intel64/bin/mpiicc
export MPIF90=/apps/compilers/intel/parallel_studio_xe_2018_update2/compilers_and_libraries/linux/mpi/intel64/bin/mpiifort
export MPIF77=/apps/compilers/intel/parallel_studio_xe_2018_update2/compilers_and_libraries/linux/mpi/intel64/bin/mpiifort

# HDF5 Paths
export HDF5_DIR="$DIR/hdf5-1.10.6"
export LDFLAGS="-L$HDF5_DIR/lib -L$DIR/netcdf-c-4.6.1/lib"
export CPPFLAGS="-I$HDF5_DIR/include -I$DIR/netcdf-c-4.6.1/include"
export LD_LIBRARY_PATH="$HDF5_DIR/lib:$DIR/netcdf-c-4.6.1/lib:$LD_LIBRARY_PATH"

# NetCDF Paths
export NCDIR="$DIR/netcdf-c-4.6.1"
export NETCDF_CFLAGS="-I$NCDIR/include -I$DIR/netcdf-c-4.6.1/include"
export NETCDF_LIBS="-L$NCDIR/lib -lnetcdf"

CFLAGS='-O2 ' CXXFLAGS='-O2 ' AM_FFLAGS='-lifcoremt ' FFLAGS='-O1 ' AM_FCFLAGS='-lifcoremt ' FCFLAGS='-O1 ' AM_LDFLAGS='-lifcoremt '

ulimit -s unlimited

```

# LIBRARIES

**hdf5**
```
wget https://support.hdfgroup.org/ftp/HDF5/releases/hdf5-1.10/hdf5-1.10.6/src/hdf5-1.10.6.tar.bz2
tar -xf hdf5-1.10.6.tar.bz2
cd hdf5-1.10.6
./configure --prefix=/home/apps/chpc/earth/delft3d/LIBRARIES/hdf5-1.10.6 --enable-shared
make
make check
make install
cd ..
```

**netcdf-c**
```
wget https://github.com/Unidata/netcdf-c/archive/refs/tags/v4.6.1.tar.gz -O netcdf-c-4.6.1.tar.gz
tar -xf netcdf-c-4.6.1.tar.gz
cd netcdf-c-4.6.1
./configure --prefix=/home/apps/chpc/earth/delft3d/LIBRARIES/netcdf-c-4.6.1 --disable-dap-remote-tests
make 
make check
make install
cd ..
```
**netcdf-fortran**
```
wget https://github.com/Unidata/netcdf-fortran/archive/refs/tags/v4.5.0.tar.gz -O netcdf-fortran-4.5.0.tar.gz
tar -xf netcdf-fortran-4.5.0.tar.gz
cd netcdf-fortran-4.5.0
./configure --prefix=/home/apps/chpc/earth/delft3d/LIBRARIES/netcdf-fortran-4.5.0
make 
make check 
make install 
cd ..
```
 - If the **netcdf-fortran** build fails, regenerate the build files and run ./configure again:
```
aclocal
autoconf
automake --add-missing
```
**Install Delft3D**: 
With all the necessary dependencies installed, we can now download, compile and install Delft3D. Note that you must have a Deltares SVN server account to download the source code. If you need to register, see the “Steps needed to use the Delft3D source code” section of the Delft3D compilation guide. In the compilation instructions below, replace <username> and <password> with the SVN login details that Deltares sent you. The commands below will install tag 68819, the recommended tag for Delft3D-FM. This time the binaries are placed /home/apps/chpc/earth/Delft3D/bin directory.
```
svn checkout --username <username> --password <password> https://svn.oss.deltares.nl/repos/delft3d/tags/delft3dfm/68819/ delft3dfm-68819
cp delft3dfm-68819/src/third_party_open/swan/src/*.[fF]* delft3dfm-68819/src/third_party_open/swan/swan_mpi
cp delft3dfm-68819/src/third_party_open/swan/src/*.[fF]* delft3dfm-68819/src/third_party_open/swan/swan_omp
cd delft3dfm-68819/src
./autogen.sh --verbose 
./configure --prefix=/home/apps/chpc/earth/delft3d/delft3d
make ds-install 
make ds-install -C engines_gpl/dflowfm 
cd ..
```
# Test Delft3D
To test a structured mesh example, edit the run.sh file appropriately and issue the following commands:
```
pushd examples/01_standard
./run.sh
popd
```

To test a flexible mesh example, issue the following:
```
pushd examples/12_dflowfm/test_data/e02_f14_c040_westerscheldt
./run.sh
popd
```
