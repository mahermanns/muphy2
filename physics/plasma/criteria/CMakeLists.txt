target_include_directories(muphy2 PRIVATE .)
target_sources(muphy2 PRIVATE
    criterion_j.cc
    criterion_j.h
    criterion_j_ue.cc
    criterion_j_ue.h
    criterion_position.cc
    criterion_position.h
    load_balancing.cc
    load_balancing.h
    select_scheme.cc
    select_scheme.h
    )
