### json格式的数据返回

```java
@Override
public List<LayerDirectoryVo> getLayerTree() {
    List<LayerDirectoryRelation> list = layerDirectoryRelationService.list();
    List<LayerDirectoryVo> tree = new ArrayList<>();
    Map<String, List<LayerDirectoryRelation>> layerDirectoryRelationMap = list.stream()
            .collect(Collectors.groupingBy(LayerDirectoryRelation::getLayerDirectoryId));

    layerDirectoryRelationMap.forEach((directoryId, layerRelationList) -> {
        LayerDirectoryVo layerDirectoryVo = new LayerDirectoryVo();
        LayerDirectory layerDirectory = this.baseMapper.selectById(directoryId);
        BeanUtils.copyProperties(layerDirectory, layerDirectoryVo);
        List<String> layerIds = layerRelationList.stream().map(LayerDirectoryRelation::getMapLayerId)
                .distinct().collect(Collectors.toList());
        List<MapLayerVo> layers = mapLayerService.findByLayerIds(layerIds).stream().map(m -> {
            MapLayerVo mapLayerVo = new MapLayerVo();
            BeanUtils.copyProperties(m, mapLayerVo);
            return mapLayerVo;
        }).collect(Collectors.toList());
        layerDirectoryVo.setMapLayerVoList(layers);
        tree.add(layerDirectoryVo);
    });

    return tree;
}
```

根据中间表封装信息

```java
private void attachMember(List<SysRoleVO> list) {
        Set<Long> roleIds = list.stream().map(SysRoleVO::getId).collect(Collectors.toSet());
        List<SysRoleUser> sysRoleUsers = roleUserService.queryForArrayList(roleIds,"role_id");
        Set<Long> userIds = sysRoleUsers.stream().map(SysRoleUser::getSysUserId).collect(Collectors.toSet());
        ArrayList<SysUser> sysUsers = userService.queryForArrayList(userIds, "id");
        Map<Long, SysUser> userMap = sysUsers.stream().collect(Collectors.toMap(SysUser::getId, sysUser -> sysUser));
        Map<Long, List<Long>> roleToUserIdsMap = sysRoleUsers.stream()
                .collect(Collectors.groupingBy(
                        SysRoleUser::getRoleId,
                        Collectors.mapping(SysRoleUser::getSysUserId, Collectors.toList())
                ));
        for (SysRoleVO vo : list) {
            List<Long> userIdList = roleToUserIdsMap.getOrDefault(vo.getId(), Collections.emptyList());
            if (userIdList.isEmpty()) {
                vo.setMembers("");
                continue;
            }
            String names = userIdList.stream().map(userMap::get).filter(Objects::nonNull)
                    .map(SysUser::getName)
                    .collect(Collectors.joining(","));
            vo.setMembers(names);
        }
    }
```

