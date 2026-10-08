# Jujutsu-kernel Development & Integration Guidelines

## Target Platform & Build Invariants
- **Device**: Xiaomi Redmi 9A / 9C / 9A NFC (`blossom` / `dandelion` / `angelica` / `cattail`, MediaTek MT6762 / MT6765).
- **Architecture**: `arm64` (`ARCH=arm64`, `SUBARCH=arm64`).
- **Userspace / ROM**: Pure 64-bit (`arm64`) custom ROMs (AOSP/LineageOS-based).
- **MIUI Compatibility**: Strictly **INCOMPATIBLE** with MIUI. Do not add workarounds, shims, or spend effort accommodating MIUI proprietary blobs/services.
- **Kernel Version**: Linux 4.19.275.
- **Defconfig**: `arch/arm64/configs/blossom_defconfig`.
- **Active Branch**: `V7` (SUSFS v2.3.0 with Universal Self-Healing). `V7-backup` is the pre-v2.3.0 archive.
- **Known Broken Configs**:
  - `CONFIG_USERFAULTFD`: Must be disabled (`# CONFIG_USERFAULTFD is not set` / `CONFIG_USERFAULTFD=n`). Backported userfaultfd structures (`sysctl_unprivileged_userfaultfd`, `VM_UFFD_MINOR`) are incomplete in this tree and break `kernel/sysctl.c` and `fs/proc/task_mmu.c`.

## BakaSU (formerly ReSukiSU) + SUSFS Integration Rules
When developing, patching, or updating BakaSU and SUSFS in this kernel tree:

### 1. Non-GKI (4.19) Hook Signature Invariant
- **Issue**: On GKI (5.10+), `do_faccessat` and `vfs_statx` pass `struct filename *`. On Non-GKI 4.19, `fs/open.c` and `fs/stat.c` pass `const char __user *filename`.
- **Rule**: BakaSU `kernel/feature/sucompat.h` and `sucompat.c` MUST NOT use `struct filename **filename` on kernel 4.19. Always gate `struct filename **` with `LINUX_VERSION_CODE >= KERNEL_VERSION(5, 10, 0)` so 4.19 uses `const char __user **filename_user`.
- **Early Boot Protection**: Always guard calls to `ksu_handle_faccessat` and `ksu_handle_stat` with `if (likely(current->mm))` in `fs/open.c` and `fs/stat.c` so kernel threads (e.g., `swapper/0` checking `/init`) never invoke userspace sucompat hooks.

### 2. `include/linux/susfs_def.h` Protocol Requirements
BakaSU relies on `susfs_def.h` for process privilege checking and unmounting. Ensure all corresponding thread flags and inline helpers exist:
- Thread Info Flags:
  - `TIF_PROC_UMOUNTED (33)`
  - `TIF_PROC_NO_SU (34)`
  - `TIF_PROC_UMOUNTED_FOR_ZYGOTE_NEXT (35)`
- Required inline helpers:
  - `susfs_is_current_proc_no_su(void)` -> `test_ti_thread_flag(&current->thread_info, TIF_PROC_NO_SU)`
  - `susfs_set_current_proc_no_su(void)` -> `set_ti_thread_flag(&current->thread_info, TIF_PROC_NO_SU)`
  - `susfs_clear_current_proc_no_su(void)` -> `clear_ti_thread_flag(&current->thread_info, TIF_PROC_NO_SU)`
  - `susfs_is_current_proc_umounted(void)`
  - `susfs_set_current_proc_umounted(void)`
  - `susfs_clear_current_proc_umounted(void)`
  - `susfs_is_current_proc_umounted_for_zygote_next(void)`
  - `susfs_set_current_proc_umounted_for_zygote_next(void)`
  - `susfs_clear_current_proc_umounted_for_zygote_next(void)`

### 3. Variable Typing: `bool` with `READ_ONCE()` vs. `static_key`
- In `fs/susfs.c`, control flags (`susfs_hide_sus_mnts_for_non_su_procs`, `susfs_is_avc_log_spoofing_enabled`) are declared as `bool`.
- DO NOT use `static_branch_unlikely()` or `extern struct static_key_false` on these variables. Always declare them as `extern bool` and access them with `READ_ONCE()`.
- Ensure `kfree(scontext)` is called in `security/selinux/avc.c` when context spoofing occurs to avoid kernel memory leaks.

### 4. Patch Deduplication & Deprecated Symbols
- Never apply partial or upstream SUSFS diffs without verifying against existing hooks.
- Check for duplicate `orig_flow:` labels in `fs/readdir.c`.
- In `fs/stat.c`, use `susfs_sus_kstat_spoof_generic_fillattr(inode, stat)` (defined in `fs/susfs.c`). Do NOT use `susfs_sus_ino_for_generic_fillattr` as it is non-existent.
- Do NOT duplicate extern declarations or hook blocks in `fs/proc/base.c` and `fs/proc/task_mmu.c`.

### 5. SUSFS v2.3.0 Non-GKI (4.19) Adaptation Invariants
When maintaining or backporting SUSFS v2.3.0 features to this 4.19 kernel:
- **`generic_fillattr` Hook Signature**:
  - GKI 6.1 upstream passes 3 arguments (`inode, stat, result_mask`). On Non-GKI 4.19, `generic_fillattr` only has 2 arguments (`inode, stat`).
  - `susfs_sus_kstat_spoof_generic_fillattr` in `fs/susfs.c` and `fs/stat.c` MUST remain 2 arguments: `(struct inode *inode, struct kstat *stat)`.
  - Gate `stat->mnt_id` with `#if LINUX_VERSION_CODE >= KERNEL_VERSION(5, 10, 0)` as `mnt_id` does not exist in 4.19 `struct kstat`.
- **`fs/statfs.c` Exported Wrappers**:
  - `fs/susfs.c` requires `statfs_by_dentry_wrapper` and `calculate_f_flags_wrapper`.
  - Export these wrappers as non-static functions in `fs/statfs.c` guarded by `#ifdef CONFIG_KSU_SUSFS_SUS_KSTAT`.
- **`vfs_statfs` Spoofing Guards**:
  - In `fs/statfs.c:vfs_statfs`, guard `susfs_statfs_by_dentry` with `susfs_is_current_app_uid()` and `susfs_is_inode_sus_kstat()`.
- **Memory Leak Protection**:
  - In `susfs_update_sus_kstat()`, always call `kfree(new_entry)` if no existing matching entry is found before returning `-ENOENT`.
- **Typo Invariant**:
  - Ensure `#define KSTAT_SPOOF_CTIME_TV_SEC (1 << 8)` uses `<<` and `is_statically` is typed as `bool` in `include/linux/susfs.h`.

### 6. Low-RAM Inode Eviction & Universal Self-Healing Invariants
On low-RAM devices (e.g. 2GB MT6762 / Redmi 9A), unreferenced inodes with `refcount == 0` are frequently evicted from dcache/icache by `shrink_icache_sb` during app cold launches, Termux commands, and background memory pressure:
- **Fast-path Self-Healing Architecture**:
  - `inode->i_mapping->flags` bits (`AS_FLAGS_SUS_PATH`, `AS_FLAGS_OPEN_REDIRECT`, `AS_FLAGS_SUS_KSTAT`, `AS_FLAGS_SUS_MAP`) are purely in-memory flags that are wiped when an inode is evicted and reloaded from storage.
  - When an app or command accesses an evicted path, Linux loads a fresh `struct inode` from disk with `flags == 0`.
  - To prevent detection failure, every feature implements a 3-tier check:
    1. **Tier 1 (1 CPU instruction)**: `test_bit(AS_FLAGS_*, &inode->i_mapping->flags)` — zero overhead for cached inodes.
    2. **Tier 2 (Zero-overhead check)**: `if (likely(hash_empty(FEATURE_HLIST))) return false;` — instantaneous bailout if no paths configured.
    3. **Tier 3 (Self-Healing via RCU Hash Table)**: Lookup by permanent disk identifiers `(inode->i_ino, inode->i_sb->s_dev)` under `rcu_read_lock()`. On match, re-arm the flag (`set_bit(AS_FLAGS_*, &inode->i_mapping->flags)`) on-the-fly so subsequent accesses return to Tier 1 fast-path!
- **Component Implementations**:
  - **`sus_path`**:
    - Backed by `SUS_PATH_HLIST`. Enrolled dynamically in `susfs_add_sus_path()`, `susfs_add_sus_path_loop()`, and `susfs_run_sus_path_loop()`.
    - `susfs_is_inode_sus_path(inode)`: Self-heals both standard and FUSE inodes upon access in `fs/namei.c`.
    - `susfs_is_ino_sus_path(dev, ino)`: Fallback check in `fs/readdir.c` callbacks when `ilookup(buf->sb, ino)` returns `NULL` (inode not in icache during directory listing).
  - **`open_redirect`**:
    - `susfs_is_inode_open_redirect_fast(inode)`: Backed by `OPEN_REDIRECT_HLIST` under SRCU.
    - `susfs_run_open_redirect_loop()`: Enrolled in `susfs_run_extra_works()`.
  - **`sus_kstat`**:
    - `susfs_is_inode_sus_kstat(inode, out_is_fuse)`: Backed by `SUS_KSTAT_HLIST`. Self-heals inode flag, protecting `generic_fillattr()`, `vfs_statfs()`, and `show_map_vma()`.
  - **`sus_map`**:
    - `susfs_is_inode_sus_map_fast(inode)`: Backed by `SUS_MAP_HLIST`. Self-heals `AS_FLAGS_SUS_MAP` protecting `/proc/<pid>/maps` and `/proc/<pid>/smaps` filtering.
- **Cold Path & Eviction Interception (`fs/namei.c` & `fs/open.c`)**:
  - Upstream SUSFS only intercepted cached dentries in `lookup_fast` / `d_lookup`. When an inode was evicted or on cold start, `d_lookup` returned `NULL`, causing Linux to take the `atomic_open` (ext4/f2fs) / `lookup_open` / `may_open` paths which lacked hooks, allowing the first read to succeed before the inode was cached!
  - To ensure cold reads never leak, `susfs_is_inode_sus_path()` is hooked across the entire VFS open & lookup pipeline: `may_open()`, `atomic_open()`, `lookup_open()`, `filename_lookup()`, `do_o_path()`, and `fs/open.c:do_sys_openat()`.
  - **Pipeline Cleanup Invariants**:
    - In `atomic_open()`: When `susfs_is_inode_sus_path()` matches, call `fput(file)` (if `file->f_mode & FMODE_OPENED`), `dput(dentry)`, and return `-ENOENT`.
    - In `lookup_open()`: In the `no_open` path after `dir_inode->i_op->lookup()`, check `susfs_is_inode_sus_path(dentry->d_inode)` and jump to `out_dput` returning `-ENOENT`.
    - In `fs/open.c:do_sys_openat()`: After `do_filp_open()` returns, check `susfs_is_inode_sus_path(file_inode(f))`. If matched, call `filp_close(f, NULL)` and replace `f = ERR_PTR(-ENOENT)` so `put_unused_fd(fd)` is invoked cleanly.
    - In `filename_lookup()`: Guard resolved paths when `!(flags & LOOKUP_PARENT)` with `path_put(path)` and return `-ENOENT`.
- **Loop List Decoupling**:
  - `susfs_add_sus_path()` must NOT auto-enroll into `LH_SUS_PATH_LOOP`. Low-RAM eviction persistence is completely solved by `SUS_PATH_HLIST` self-healing, while `LH_SUS_PATH_LOOP` remains reserved for explicit userspace `add_sus_path_loop` calls.


