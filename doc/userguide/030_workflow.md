\hypertarget{workflow}{}

# Workflow \label{chap:workflow}

In this chapter, the complete process of setting up a simulation in FLEXI is detailed.

## Mesh generation using HOPR
FLEXI obtains its computational meshes solely from the high order preprocessor HOPR (available under GPLv3 at [https://www.hopr-project.org](https://www.hopr-project.org)) in HDF5 format. The design philosophy is that all tasks related to mesh organization, different input formats and the construction of high order geometrical mappings are separated from the *parallel* simulation code. These tasks are implemented most efficiently in a *serial* environment.

The employed mesh format is designed to make the parallel read-in process as simple and fast as possible. For details concerning the mesh format please refer to the [HOPR HDF5 Curved Mesh Format documentation](https://www.hopr-project.org/upload/e/e6/MeshFormat.pdf).

Using HOPR, simple, structured meshes can be directly created using an inbuilt mesh generator. More complex geometries can be treated by importing meshes generated by external mesh generators in CGNS or GMSH format. A number of strategies to create curved boundaries are also included in HOPR.

The test cases provided in Chapter \ref{chap:tutorials} come with both a ready to use mesh file as well as a parameter file for HOPR, which can be used to generate or modify the meshes as needed.

Provided the mesh file has been set up, its location must be specified in the parameter file.

    MeshFile=[path to mesh file.h5]

    
## Solver settings

Before setting up a simulation, the code must be compiled with the desired parameters. The most important compiler options to be set are

* ``FLEXI_EQNSYSNAME``, e.g. *Navier-Stokes*
* ``FLEXI_NODETYPE``, the nodal collocation points used during the simulation. Available options are either GAUSS or GAUSS-LOBATTO.

All other options are set in the parameter file. The most important steps are

* **Set the polynomial degree ``N``**

    Defines the polynomial degree of the solution. The order of convergence follows as $N+1$. Each grid cell contains $(N+1)^3$ collocation points to represent the solution.
* **Choose a de-aliasing approach.**
    
    For under-resolved Navier-Stokes simulations, e.g. in an LES setting, de-aliasing is important for numerical stability. Various choices are available and set using ``OverintegrationType``. 
    
    * ``OverintegrationType=1``
    
         The first option is a filtering strategy. The complete operator is first evaluated at ``N`` ($U_t^{N}$) and then filtered to a lower effective degree ``NUnder`` ($U_t^{Nunder}$).
    
         To use this variant, specify ``Nunder`` to a value smaller than ``N``.
        
    * ``OverintegrationType=2``
    
         In this variant of the first option, the operator in reference space, e.g. $JU_t$, is first projected to the ``NUnder`` node set before converting it to physical space $U^{Nunder}_t=JU^{Nunder}_t/J^{Nunder}$. This implementation enforces conservation.
    
         To use this variant, specify ``Nunder`` to a value smaller than ``N``.
        
    * ``OverintegrationType=3``
    
        The third option is to compute the predominantly aliasing-afflicted advective fluxes at a node set of higher polynomial degree ``NOver`` and subsequently project their contribution to the operator to the polynomial space associated with ``N``. The Lifting procedure and the viscous fluxes are still computed at the original nodal set using ``N``. 
    
        To use the advective flux over-integration, ``NOver`` must be specified.

* **Choose a Riemann solver**

    The Riemann solver defines how inter-element coupling is accomplished. The available variants are listed in Section \ref{sec:parameterfile}. Use the ``Riemann`` and the ``RiemannBC`` options to specify which Riemann solver is to be used at internal interfaces and at Dirichlet boundary conditions, respectively. The default Riemann solver is "Roe with entropy fix". 

* **Choose a time discretization method** 

    The time discretization method is set using the option ``TimeDiscMethod``. Various explicit Runge-Kutta variants are available and listed in Section \ref{sec:parameterfile}. By default, the low-storage fourth order Runge-Kutta scheme by [@Carpenter1994] is employed.


## Initial and boundary conditions

Initial and boundary conditions are controlled via the so-called ``RefState`` and ``ExactFunction`` constructs.

The ``RefState`` basically specifies a state vector in primitive form $(\rho,u,v,w,p)^T$. An arbitrary number of ``RefState``s can be defined:

    RefState=(/1,1,0,0,0.71428571/)
    RefState=(/1,0.3,0,0,0.71428571/)

In this example, the first state would result in a parallel flow in $x$ direction at $Ma=1$, the second state at $Ma=0.3$.
    


### Initial conditions

The code contains a number of pre-defined analytic solution fields (ExactFunctions), which are invoked by specifying their respective number. For instance the initialization of a simple constant freestream is achieved by setting

    IniExactFunc=1

The associated state vector to be used is determined by

    IniRefState=1

This implies that the first of the two available ``RefState``s is used for initialization. 

Note: currently, the ExactFunctions contained in the code are not documented yet. They can be looked up in the source file ``src/equations/navierstokes/equation.f90``.

### Boundary conditions \label{sec:boundaryconditions}

\hypertarget{target:sec:boundaryconditions}{}
The names of the boundaries are contained in the mesh file and can be used in the Flexi parameter file to override the boundary conditions already set in the parameter file, if necessary.

FLEXI lists the boundaries and their respective boundary conditions during initialization:

~~~~~~~~~~~~~~~~~~~{.bash}
|                Name      Type     State     Alpha
|       BC_periodicz-         1         0         3
|       BC_periodicy-         1         0         2
|       BC_periodicx+         1         0        -1
|       BC_periodicy+         1         0        -2
|       BC_periodicx-         1         0         1
|       BC_periodicz+         1         0        -3
~~~~~~~~~~~~~~~~~~~

Suppose, we wish to apply a Dirichlet boundary condition with ``RefState`` 2 at the two lateral boundaries. Therefor, we have to add the following lines to the parameter file

    BoundaryName=BC_periodicy-
    BoundaryType=(/2,2/)
    BoundaryName=BC_periodicy+
    BoundaryType=(/2,2/)
    
Note that the first index within brackets specifies ``BC_TYPE``, while the second one specifies ``BC_STATE``, in this case the number of the ``RefState`` to be used. In general, ``BC_STATE`` identifies either a ``RefState``, an ``ExactFunction`` or remains empty, dependent on the ``BC_TYPE``. Currently implemented boundary types for *Navier-Stokes* are listed in table \ref{tab:boundaryconditions}.


 Boundary Condition |BC_TYPE          |BC_STATE           |Comment                   
|:-----------------:|:---------------:|:-----------------:|:-------------------------------|
|  Periodic BC      |  1              |       -           | Can only be defined in HOPR    |
|  Weak Dirichlet   |  2              | ``RefState``      |                                |
|  Weak Dirichlet   |  12             |     -             | Like 2, but using an external  |
|                   |                 |                   | state set by ``BCStateFile``   |
|  Weak Dirichlet   |  22             | ``ExactFunction`` | Like 2, but using an           |
|                   |                 |                   | ``ExactFunction``              |
|  Wall adiabatic   |  3              |     -             |                                |
|  Wall isothermal  |  4              | ``RefState``      | Isothermal wall, temperature is|
|                   |                 |                   | specified via $p$ and $\rho$   |
|                   |                 |                   | contained in the ``RefState``  |
|  Wall slip        |  9              |     -             | Slip, symmetry or Euler wall   |
|  Outflow Mach no. |  23             | ``RefState``      | [^1]                           |
|  Outflow Pressure |  24             | ``RefState``      |                                |
|  Outflow Subsonic |  25             | ``RefState``      |                                |
|  Inflow Total pressure / Temp.   |  27             | ``RefState``      | **Special Refstate:** *total* quantities          |
|                   |                 |                   | $(T_t,\alpha,\beta,0,p_t)$     |

Table: Boundary conditions. \label{tab:boundaryconditions}

[^1]: see [@carlson2011inflow] for details on the listed inflow/outflow boundary conditions.

## Material properties

At present, the only available equation of state in the *Navier-Stokes* solver of FLEXI is the ideal gas. The gas constant, adiabatic exponent, Prandtl number and viscosity are specified in the parameter file using ``R``, ``kappa``, ``Pr`` and ``mu0``.

## Output time interval

Set the end time of the simulation using ``TEnd`` and the interval in which the solution is dumped to the hard drive with ``Analyze_dt``. 

Note that evaluation of body forces and other runtime analysis routines are also invoked once in every analyze interval determined by ``Analyze_dt``. Set e.g. ``nWriteData=10`` to a value greater one to restrict the solution output to every 10th ``Analyze_dt``.

### Restart the simulation {-}

The simulation may be restarted from an existing state file

    $FLEXI_DIR/flexi parameter.ini [restart_file.h5]
    
**Note: when restarting from an earlier time (or zero), all later state files possibly contained in your directory are deleted!**

## Evaluation during runtime

At every ``Analyze_dt``, the following evaluations are possible:

* ``CalcErrorNorms=T``: Calculate the $L_2$ and $L_\infty$ error norms based on the specified ``ExactFunc`` as reference. This evaluation is used for e.g. convergence tests.

* ``CalcBodyForces=T``: Calculate the pressure and viscous forces acting on every wall boundary condition (BC_TYPE=3,4 or 9) separately. The forces are written to .dat files.

* ``CalcBulkState=T``: Calculate the bulk quantities (e.g. bulk velocity in channel flow).

* ``CalcWallVelocity=T``: Due to the discontinuous solution space and the weakly enforced boundaries, the no-slip condition is not exactly fulfilled. The deviation depends mainly on the resolution in the near-wall region. Thus, this evaluation can be used as a resolution measure at the wall.

## Parallel execution
The simulation code is specifically designed for (massively) parallel execution using the MPI library. For parallel runs, the code must be compiled with `FLEXI_MPI=ON`.

Parallel execution is then controlled using `mpirun` \label{missing:flexi_variable_notset}

    mpirun -np [no. processors] $FLEXI_DIR/flexi parameter.ini
    
### Domain decomposition

The grid elements are organized along a space-filling curved, which gives a unique one-dimensional element list. In a parallel run, the mesh is simply divided into parts along the space filling curve. Thus, domain decomposition is done *fully automatic* and is not limited by e.g. an integer factor between the number of cores and elements. The only limitation is that the number of cores may not exceed the number of elements.

### Choosing the number of cores
Parallel performance heavily depends on the number of processing cores. The performance index is defined as

(@PID) $$PID=\frac{WallTime}{nCores\cdot nDOF \cdot nTimeSteps}$$

and measures the CPU time per degree of freedom and time step. During runtime, the average $PID$ is displayed in the output

~~~~~~~~~~~~~~~~~~ {.Bash}
 CALCULATION TIME PER STAGE/DOF: [ 5.59330E-07 sec ]
~~~~~~~~~~~~~~~~~~

When compared to the single performance, it can be used as a parallel efficiency measure. The $PID$ is mainly dependent on the load per core

(@Load) $$Load=nDOF/nCores$$

and the polynomial degree $N$. Load values for optimal performance lie in the range $Load=2000-5000$. A detailed parallel performance analysis at the example of a Cray XC-40 system is given in [@atak2016high].

## Test case environment \label{sec:testcases}

The test case environment can be used as to add test case-specific code for e.g. custom source terms or diagnostics to be invoked during runtime. 

The compiler option `FLEXI_TESTCASE` sets the current test case. The test cases are contained in the *src/testcase/* folder.

Standardized interfaces are defined for initialization, source terms and analysis routines

* **InitTestcase**

    Read in testcase related parameters from the `parameter.ini`, initialize the corresponding data structures.

* **FinalizeTestcase**

    Deallocate test case specific data structures.

* **ExactFuncTestcase**

    Define test case specific analytic expressions for initial or boundary conditions.

* **CalcForcing**

    Impose test case specific source terms, e.g. the pressure gradient in test case `channel`.
 
* **AnalyzeTestCase**

    Perform test case specific diagnostics.
    
Currently supplied test cases are

* **default**
* **channel**: turbulent channel flow with steady pressure gradient source term
* **phill**: periodic hill flow with controlled pressure gradient source term \label{missing:phill_testcase}
* **taylorgreenvortex**: Automatic diagnostics for the Taylor-Green vortex flow

Note that the test case environment is currently only applicable to the *Navier-Stokes* equation system.

## Post processing / FLEXI2VTK tool \label{sec:convert_tool}

The current release of FLEXI comes with a **FLEXI2VTK** tool for visualization with Paraview. The FLEXI2VTK tool takes the HDF5 files generated by FLEXI. The *conservative* state vector is visualized. 

~~~~~~~
$FLEXI_DIR/flexi2vtk parameter.ini [flexi_out.h5]
~~~~~~~

The FLEXI2VTK tool runs in parallel with activated `FLEXI_MPI` flag \label{missing:convert_variable_notset3}

~~~~~~~
mpirun -np [no. processors] $FLEXI_DIR/flexi2vtk parameter.ini [flexi_out.h5]
~~~~~~~

Multiple HDF5 files can be passed to the FLEXI2VTK tool at once, even from different calculations with different settings.

The runtime parameters to be set in `parameter.ini` are

-------------   ---- --------------------------------------------------
NodeTypeVisu    VISU Node type of the visualization basis:  
                     VISU,GAUSS,GAUSS-LOBATTO,CHEBYSHEV-GAUSS-LOBATTO  
                      
NVisu                Polynomial degree at which solution is sampled for  
                     visualization.  
                      
useCurveds         T Controls usage of high-order information in mesh.  
                     Turn off to discard high-order data and treat  
                     curved meshes as linear meshes
-------------   ---- --------------------------------------------------

Table: Runtime parameters for the FLEXI2VTK tool.

Some of these options are duplicates from options for FLEXI. You can use the same parameter file for both executables.

The node type VISU uses equidistant nodes which include the boundary points of elements.

There is an alternative mode of usage called command line mode that does not use a parameter file. In this case NVisu is directly
specified on the command line and all other options are set to their standard values. The syntax for this mode is

~~~~~~~
mpirun -np [no. processors] $FLEXI_DIR/flexi2vtk --NVisu=INTEGER [flexi_outputfile.h5]
~~~~~~~

where INTEGER is substituted by the desired degree of the visualization basis. 