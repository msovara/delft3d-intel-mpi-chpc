# Delft3D installation on LENGAU runnning CentOS-7


Delft3D is Open Source Software and facilitates the hydrodynamic (Delft3D-FLOW module), morphodynamic (Delft3D-MOR module), waves (Delft3D-WAVE module), water quality (Delft3D-WAQ module including the DELWAQ kernel) and particle (Delft3D-PART module) modelling. The source code can be downloaded here: https://oss.deltares.nl/web/delft3d. This repo is based on the installation tutorial here: https://gist.github.com/H0R5E/c4af6db788b227de702a12e01b64cf46 and https://www.youtube.com/watch?v=o1iQvO3x8A0. 

2018 Intel OneAPI compilation on CentOS-7 using Tmux inside an interactive session.

# Delft3D Compilation Guide

## Prerequisites
- Intel OneAPI Compiler Suite
- MPICH 4.2.2
- Access to CHPC modules
- Sufficient disk space (~2GB)


## 1. Environment Setup

### Create Environment Script
Create a file named `delft3d_env.sh`:
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

# Load Intel OneAPI environment
INTEL_SETVARS="/home/apps/chpc/compmech/compilers/intel/oneapi/setvars.sh"
if [ -f "$INTEL_SETVARS" ]; then
    source "$INTEL_SETVARS" intel64 || {
        echo "Error: Failed to source Intel environment"
        exit 1
    }
else
    echo "Error: Intel setvars.sh not found"
    exit 1
fi

# Load additional modules
module load chpc/compmech/mpich/4.2.2/oneapi2023-ssh || exit 1
module load chpc/BIOMODULES || exit 1
module add curl/7.50.0 || exit 1
module load gcc/9.2.0 || exit 1

#---------------
# Directory Setup
#---------------
export DelftDIR=/home/apps/chpc/earth/delft3d_mpich_oneapi
export DIR=${DelftDIR}/LIBRARIES
export NCDIR=${DIR}/netcdf-c-4.6.1
export NCFDIR=${DIR}/netcdf-fortran-4.5.0
export HDF5_DIR=${DIR}/hdf5-1.10.6
export MPICH_ROOT=/home/apps/chpc/compmech/mpich-4.2.2-oneapi2023

#---------------
# Compiler Setup
#---------------
print_section "Compiler Setup"

# Use Intel compilers directly
export CC=icc
export CXX=icpc
export FC=ifort
export F77=ifort

# MPI wrappers
export MPICC="$MPICH_ROOT/bin/mpicc"
export MPICXX="$MPICH_ROOT/bin/mpicxx"
export MPIFC="$MPICH_ROOT/bin/mpif90"
export MPIF77="$MPICH_ROOT/bin/mpif77"

# Force Intel compilers for MPI
export OMPI_CC=icc
export OMPI_CXX=icpc
export OMPI_FC=ifort
export OMPI_F77=ifort

# Print compiler versions
echo "Checking compiler versions..."
$CXX --version || exit 1
$CC --version || exit 1
$FC --version || exit 1

#---------------
# Compiler and Linker Flags
#---------------
# Basic optimization flags
export CFLAGS="-O2 -fPIC"
export CXXFLAGS="-O2 -fPIC"
export FFLAGS="-O2 -fPIC"
export FCFLAGS="-O2 -fPIC"

# Linker flags
export LDFLAGS="-lifcore -lifport -lifcoremt -limf -lsvml -lm -lipgo -lirc -lpthread -lirc_s -ldl"

#---------------
# Library Paths
#---------------
print_section "Library Paths Setup"
INTEL_LIB="/home/apps/chpc/compmech/compilers/intel/oneapi/compiler/2023.2.0/linux/compiler/lib/intel64_lin"

export LD_LIBRARY_PATH="\
$MPICH_ROOT/lib:\
$INTEL_LIB:\
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
#export FFLAGS=${FCFLAGS}
#export CFLAGS=${FCFLAGS}
#export CXXFLAGS="-O2 -fPIC"
export AM_FFLAGS='-lifcoremt'
export AM_LDFLAGS='-lifcoremt'

print_section "Compiler Flags Setup"

# Basic flags
export CFLAGS="-O2 -fPIC"
export CXXFLAGS="-O2 -fPIC"
export FFLAGS="-O2 -fPIC -assume noold_unit_star"
export FCFLAGS="-O2 -fPIC -assume noold_unit_star"

# Linker flags with explicit Intel Fortran libraries
export LDFLAGS="\
-L$MPICH_ROOT/lib \
-L$INTEL_LIB \
-L${NCDIR}/lib \
-L${HDF5_DIR}/lib"

# Additional libraries needed for linking
export LIBS="\
-lifcore -lifport -lifcoremt -limf -lsvml -lm -lipgo -lirc -lpthread -lirc_s -ldl"

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

## 2. Build HDF5
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

## 3. Building netcdf-c
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

## 4. Building netcdf-fortran
```
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

# Delft3D-FM Installation Guide

## Prerequisites

- SVN account from Deltares
- Completed installation of HDF5, NetCDF-C, and NetCDF-Fortran
- Sourced environment from `delft3d_env.sh`
```
set -e  # Exit on error

#---------------
# Configuration
#---------------
DELFT3D_VERSION="68819"
INSTALL_PREFIX="/home/apps/chpc/earth/delft3d_mpich_oneapi/delft3d"
SVN_URL="https://svn.oss.deltares.nl/repos/delft3d/tags/delft3dfm/${DELFT3D_VERSION}/"
SOURCE_DIR="delft3dfm-${DELFT3D_VERSION}"qsubi

#---------------
# Main Installation
#---------------
main() {
    print_section "Checking Prerequisites"
    check_prerequisites

    print_section "Downloading Delft3D Source Code"
    # Prompt for credentials if not provided
    read -p "Enter SVN username: " SVN_USER
    read -s -p "Enter SVN password: " SVN_PASS
    echo

    # Checkout source code
    svn checkout \
        --username "${SVN_USER}" \
        --password "${SVN_PASS}" \
        "${SVN_URL}" \
        "${SOURCE_DIR}"

    print_section "Copying SWAN Files"
    # Copy SWAN source files
    cp ${SOURCE_DIR}/src/third_party_open/swan/src/*.[fF]* \
       ${SOURCE_DIR}/src/third_party_open/swan/swan_mpi
    cp ${SOURCE_DIR}/src/third_party_open/swan/src/*.[fF]* \
       ${SOURCE_DIR}/src/third_party_open/swan/swan_omp

    print_section "Configuring Build"
    cd ${SOURCE_DIR}/src

# Clean Build
#---------------
print_section "Cleaning Build Directory"
rm -f config.cache config.status config.log
rm -rf autom4te.cache
find . -name "*.o" -delete
find . -name "*.a" -delete
find . -name "*.so" -delete
find . -name "*.la" -delete
find . -name "*.lo" -delete
find . -name ".deps" -exec rm -rf {} +
find . -name ".libs" -exec rm -rf {} +

    # Generate build system files
    echo "Running autogen..."
    ./autogen.sh --verbose

    # Configure build
    echo "Running configure..."
    ./configure --prefix="${INSTALL_PREFIX}" \
               --with-mpi \
               --with-netcdf \
               --with-hdf5

    print_section "Building Delft3D"
    # Main compilation
    echo "Building main components..."
    make ds-install

    echo "Building D-Flow FM..."
    make ds-install -C engines_gpl/dflowfm

    print_section "Verifying Installation"
    # Check if critical binaries exist
    for binary in d_hydro dflowfm flow2d3d; do
        if [ -f "${INSTALL_PREFIX}/bin/${binary}" ]; then
            echo "✓ ${binary} installed successfully"
        else
            echo "✗ ${binary} installation failed"
            exit 1
        fi
    done

    print_section "Installation Complete"
    echo "Delft3D has been installed to: ${INSTALL_PREFIX}"
    echo "Add the following to your PATH:"
    echo "export PATH=${INSTALL_PREFIX}/bin:\$PATH"
}

#---------------
# Utility Functions
#---------------
print_section() {
    echo -e "\n=== $1 ==="
}

#---------------
# Script Execution
#---------------
main "$@"
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
