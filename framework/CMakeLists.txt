target_include_directories(muphy2 PRIVATE .)
target_sources(muphy2 PRIVATE
    block.h
    criterion.h
    definitions.h
    interpolate.h
    model.h
    mpi_boundary.h
    output_write.h
    parameter.h
    restart.h
    scheme.h
    utilities.h
    block.cc
    definitions.F90
    interpolate.F90
    interpolate.cc
    main.cc
    mpi_boundary.F90
    mpi_boundary.cc
    output_write.cc
    parameter.cc
    restart.cc
    utilities.F90
    utilities.cc
    )

