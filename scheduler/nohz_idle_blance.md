## nohz_idle_balance

1. patch 1
    + sched: Improve load balancing in the presence of idle CPUs [ d4573c3e1c992668f5dcd57d1c2ced56ae9650b9  ]
    + https://lore.kernel.org/lkml/20150326130014.21532.17158.stgit@preeti.in.ibm.com/
2.  patch 2
    + sched/core: Convert nohz_flags to atomic_t [ a22e47a4e3f5a9e50a827c5d94705ace3b1eac0b ]
    + sched/fair: Add NOHZ_STATS_KICK [ b7031a02ec753bf9b52a94a966b05e1abad3b7a9 ]
    + sched/fair: Restructure nohz_balance_kick() [ 4550487a993d579c7329bb5b19e516d36800c8bf ]
    + sched/fair: Add NOHZ stats balancing [ a4064fb614f83c0a097c5ff7fe433c4aa139c7af ]
    + sched/fair: Update blocked load from NEWIDLE [ e022e0d38ad475fc650f22efa3deb2fb96e62542 ]
    + sched/nohz: Clean up nohz enter/exit [ 00357f5ec5d67a52a175da6f29f85c2c19d59bc8 ]
    + https://lore.kernel.org/lkml/20171221102139.177253391@infradead.org/

3. patch 3
    + sched/nohz: Stop NOHZ stats when decayed [ f643ea2207010db26f17fca99db031bad87c8461 ]
    + sched/fair: Reduce the periodic update duration [ 1936c53ce8c8d4555e9ccad2dc8d98e0637b11f7 ]
    + sched/nohz: Optimize nohz_idle_balance() [ 63928384faefba1b31c3bb77361965715a9fc71c ]
    + sched/fair: Move rebalance_domains() [ af3fe03c562055bc3c116eabe73f141ae31bf234 ]
    + sched/nohz: Merge CONFIG_NO_HZ_COMMON blocks [ dd707247ababb685ac4b8b2c6a7bf2923725e6ac ]
    + sched/fair: Move idle_balance() [ 47ea54121e46a685aa2320df8b0f71aaeedff23f ]
    + sched/fair: Update blocked load when newly idle [ 31e77c93e432dec79c7d90b888bbfc3652592741 ]
    +  http://lkml.kernel.org/r/1518622006-16089-4-git-send-email-vincent.guittot@linaro.org
