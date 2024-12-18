# Delft3D installation on LENGAU runnning CentOS-7


Delft3D is Open Source Software and facilitates the hydrodynamic (Delft3D-FLOW module), morphodynamic (Delft3D-MOR module), waves (Delft3D-WAVE module), water quality (Delft3D-WAQ module including the DELWAQ kernel) and particle (Delft3D-PART module) modelling. The source code can be downloaded here: https://oss.deltares.nl/web/delft3d. This repo is based on the installation tutorial here: https://gist.github.com/H0R5E/c4af6db788b227de702a12e01b64cf46 and https://www.youtube.com/watch?v=o1iQvO3x8A0. 

2018 Intel MPI compilation on CentOS-7

Using Tmux inside an interactive session,

## Set compile and runtime environment variables for Delf3D in a script to be sourced during compilation and execution
**The set script:** Run the bash file that sets the delft3d environment. The file needs to be made executable and sourced to apply the changes. 
```
#!/bin/bash

#===============================
# Delft3D Environment Setup
#===============================

# Function to check if directory exists
check_dir() {
    if [ ! -d "$1" ]; then
        echo "Error: Directory $1 does not exist!"
        return 1
    fi
}

# Function to validate compiler availability
check_compiler() {
    if ! command -v "$1" &> /dev/null; then
        echo "Error: Compiler $1 not found!"
        return 1
    fi
}

#---------------
# Module Setup
#---------------
echo "Setting up modules..."
module purge

# Load required modules
source /home/apps/chpc/compmech/compilers/intel/oneapi/setvars.sh || exit 1
module load chpc/compmech/mpich/4.2.2/oneapi2023-ssh
module load chpc/BIOMODULES
module add curl/7.50.0

#---------------
# Directory Setup
#---------------
export DelftDIR=/home/apps/chpc/earth/delft3d_mpich_oneapi
export DIR=${DelftDIR}/LIBRARIES
export NCDIR=${DIR}/netcdf-c-4.6.1
export NCFDIR=${DIR}/netcdf-fortran-4.5.0
export HDF5_DIR=${DIR}/hdf5-1.10.6

# Validate directories
for d in "${DelftDIR}" "${DIR}" "${HDF5_DIR}"; do
    check_dir "$d" || exit 1
done

#---------------
# Compiler Setup
#---------------
export I_MPI_ROOT=/home/apps/chpc/compmech/mpich-4.2.2-oneapi2023
export CC=mpicc
export CXX=mpicc
export FC=mpif90
export F77=mpif77

# Validate compilers
for compiler in mpicc mpif90 mpif77; do
    check_compiler "$compiler" || exit 1
done

#---------------
# Library Paths
#---------------
# Intel compiler libraries for libstdc++ compatibility
export LD_LIBRARY_PATH="\
/opt/intel/oneapi/compiler/latest/linux/compiler/lib/intel64_lin:\
${NCDIR}/lib:\
${HDF5_DIR}/lib:\
${LD_LIBRARY_PATH}"

#---------------
# Compiler Flags
#---------------
export CPPFLAGS="-I${NCDIR}/include -I${HDF5_DIR}/include"
export LDFLAGS="-L${NCDIR}/lib -L${HDF5_DIR}/lib"
export NETCDF_CFLAGS="-I${NCDIR}/include"
export NETCDF_LIBS="-L${NCDIR}/lib -lnetcdf"
export FCFLAGS="-O2 -fPIC -I${NCDIR}/include -I${HDF5_DIR}/include"
export FFLAGS=${FCFLAGS}
export CFLAGS=${FCFLAGS}
export CXXFLAGS="-O2 -fPIC"
export AM_FFLAGS='-lifcoremt'
export AM_LDFLAGS='-lifcoremt'

# Set unlimited stack size
ulimit -s unlimited

#---------------
# Verification
#---------------
echo "=== Environment Setup Summary ==="
echo "1. Loaded Modules:"
module list
echo -e "\n2. Critical Environment Variables:"
env | grep -E "DelftDIR|DIR|NCDIR|HDF5_DIR|LD_LIBRARY_PATH|MPICC|MPIF" | sort

echo -e "\nEnvironment setup completed successfully."
```

## Build HDF5
```
set -e  # Exit on error

echo "=== Building HDF5 ==="

# Download and extract
wget https://support.hdfgroup.org/ftp/HDF5/releases/hdf5-1.10/hdf5-1.10.6/src/hdf5-1.10.6.tar.bz2
tar -xf hdf5-1.10.6.tar.bz2
cd hdf5-1.10.6

# Fix for MXM/UCX logging
export MXM_LOG_LEVEL=error
export UCX_LOG_LEVEL=error

# Configure and build
./configure \
    --enable-parallel \
    --enable-shared \
    --enable-static \
    --with-pic \
    --prefix="${HDF5_DIR}"
make -j4
make install

# Verify installation
${HDF5_DIR}/bin/h5pcc -showconfig
```

## Building netcdf-c
```
set -e

echo "=== Building NetCDF-C ==="

# Download and extract
wget https://github.com/Unidata/netcdf-c/archive/refs/tags/v4.6.1.tar.gz -O netcdf-c-4.6.1.tar.gz
tar -xf netcdf-c-4.6.1.tar.gz
cd netcdf-c-4.6.1

# Configure and build
./configure \
    CC="${CC}" \
    CFLAGS="-O2 -fPIC" \
    --disable-dap-remote-tests \
    --enable-shared \
    --enable-static \
    --with-pic \
    --with-hdf5="${HDF5_DIR}" \
    --prefix="${NCDIR}"
make -j4
make install

# Verify installation
${NCDIR}/bin/nc-config --all  
```

## Building netcdf-fortran
```
#!/bin/bash
set -e

echo "=== Building NetCDF-Fortran ==="

# Download and extract
wget https://github.com/Unidata/netcdf-fortran/archive/refs/tags/v4.5.0.tar.gz -O netcdf-fortran-4.5.0.tar.gz
tar -xf netcdf-fortran-4.5.0.tar.gz
cd netcdf-fortran-4.5.0

# Configure and build
./configure \
    --prefix="${NCFDIR}" \
    --enable-shared \
    --enable-static \
    --with-pic \
    CC="${CC}" \
    FC="${FC}" \
    F77="${F77}" \
    CPPFLAGS="-I${NCDIR}/include -I${HDF5_DIR}/include" \
    LDFLAGS="-L${NCDIR}/lib -L${HDF5_DIR}/lib" \
    LIBS="-lnetcdf -lhdf5_hl -lhdf5 -lz"
make -j4
make install

# Verify installation
${NCFDIR}/bin/nf-config --all
```
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
