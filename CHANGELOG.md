# Changelog

All notable changes to this project are documented in this file.

## 1.0.0

Released on 2026-09-12

### Breaking changes

- migrate to remotefs 1

> migrate MemoryFs to the remotefs 1 blocking contract and release version 1.0.0. Every path must be absolute, pwd and change_dir are removed, operations take a shared reference over a mutex-guarded tree, connect returns unit, transfers return owned streams that commit on finish, rename and copy move whole subtrees, and the minimum supported Rust version is 1.89.

### Added

- Breaking: migrate to remotefs 1

### Fixed

- send + sync
- bump
- Return WriteStream as WriteAndSeek

## 0.1.4

Released on 2024-10-25

### Fixed

- symlink issues

## 0.1.2

Released on 2024-10-23

### Fixed

- ver
- log

## 0.1.1

Released on 2024-10-22

### Fixed

- `with_get_uid` and `with_get_gid` constructors

## 0.1.0

Released on 2024-10-22

### Added

- remotefs memory
