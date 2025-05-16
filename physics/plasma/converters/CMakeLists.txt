target_include_directories(muphy2 PRIVATE .)
target_sources(muphy2 PRIVATE
    converters.cc
    converters.h
    converters_models.F90
    converters_output.F90
    converters_physical_quantities.F90
    converters_setup.F90
    )
