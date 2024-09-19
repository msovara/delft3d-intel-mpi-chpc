# Delft3D installation on LENGAU runnning CentOS-7


Delft3D is Open Source Software and facilitates the hydrodynamic (Delft3D-FLOW module), morphodynamic (Delft3D-MOR module), waves (Delft3D-WAVE module), water quality (Delft3D-WAQ module including the DELWAQ kernel) and particle (Delft3D-PART module) modelling. The source code can be downloaded here: https://oss.deltares.nl/web/delft3d.

2018 Intel MPI compilation on CentOS-7

Using Tmux inside an interactive session,

set script
#!/bin/bash

`module purge` 

`module load chpc/parallel_studio_xe/18.0.2/2018.2.046`

`source /apps/compilers/intel/parallel_studio_xe_2018_update2/compilers_and_libraries/linux/mpi/bin64/mpivars.sh`

`module load chpc/BIOMODULES`

`module add curl/7.50.0`

export DelftDIR=/home/apps/chpc/earth/Delft3d
export DIR=$DelftDIR/LIBRARIES

export I_MPI_SHM="off"

export CC=icc
export CXX=icc
export FC=ifort
export FCFLAGS="-m64 -I$DIR/netcdf/include -I$DIR/grib2/include"
export F77=ifort
export FFLAGS=$FCFLAGS
export CFLAGS=$FCFLAGS

export MPICC=/apps/compilers/intel/parallel_studio_xe_2018_update2/compilers_and_libraries/linux/mpi/intel64/bin/mpiicc
export MPIF90=/apps/compilers/intel/parallel_studio_xe_2018_update2/compilers_and_libraries/linux/mpi/intel64/bin/mpiifort
export MPIF77=/apps/compilers/intel/parallel_studio_xe_2018_update2/compilers_and_libraries/linux/mpi/intel64/bin/mpiifort

export LDFLAGS="-L$DIR/hdf5-1.10.6/lib"
export CPPFLAGS="-I$DIR/hdf5-1.10.6/include"

export NCDIR="$DIR/netcdf-c-4.6.1"
export LD_LIBRARY_PATH=${NCDIR}/lib:${LD_LIBRARY_PATH}
export CPPFLAGS="-I${NCDIR}/include"
export LDFLAGS="-L${NCDIR}/lib"

export NETCDF_CFLAGS="-I${DIR}/netcdf-c-4.6.1/include -I${DIR}/netcdf-c-4.6.1/include"
export NETCDF_LIBS="-L${DIR}/netcdf-c-4.6.1/lib -lnetcdf"

CFLAGS='-O2 ' CXXFLAGS='-O2 ' AM_FFLAGS='-lifcoremt ' FFLAGS='-O1 ' AM_FCFLAGS='-lifcoremt ' FCFLAGS='-O1 ' AM_LDFLAGS='-lifcoremt '

ulimit -s unlimited
