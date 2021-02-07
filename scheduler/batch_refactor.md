## sched related patches from 4.14.105 => 4.18

1. patch 1
    + sched/fair: Add comment to calc_cfs_shares() [ cef27403cbe98ebda0a32d43128dd60c309eb966 ]
    + 
2. patch 2
    + sched/fair: Cure calc_cfs_shares() vs. reweight_entity() [ 3d4b60d3e3dde6ea24e439000eb3b71078da81f1 ]

3. patch 3
    + sched/fair: Remove se->load.weight from se->avg.load_sum [ c7b50216818ef3dca14a52e3499750fbad2d9691 ]
    + https://lore.kernel.org/patchwork/patch/827583/
4. patch 4
    + sched/fair: Move enqueue migrate handling [ b382a531b9fece229c8358a9fb79431cddcf93c2 ]