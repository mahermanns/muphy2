target_include_directories(muphy2 PRIVATE .)
target_sources(muphy2 PRIVATE
    fluid10.cc
    fluid10.h
    fluid5.cc
    fluid5.h
    fluid_10mom_cweno.F90
    fluid_5mom_cweno.F90
    fluid_mhd_cweno.F90
    maxwell.cc
    maxwell.h
    maxwell_fdtd.F90
    mhd.cc
    mhd.h
    ohm.F90
    ohm.cc
    ohm.h
    poisson.F90
    poisson.cc
    poisson.h
    vlasov.cc
    vlasov.h
    vlasov_lagrange5.F90
    vlasov_pfc.F90
    )
