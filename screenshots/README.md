# Capturas de pantalla — VPN Hub-and-Spoke DMVPN Fase 2 con IKEv1

Capturas del laboratorio en orden de demostración.

| # | Archivo de Evidencia | Descripción Técnica Detallada |
|---|---|---|
| 1 | [`01_topologia.png`](/screenshots/01_topologia.png) | Topología funcional en PNETLab con nombre completo y matrícula (`20250737`) visibles, los tres routers, switches y PCs encendidos. |
| 2 | [`02_config_hub_tunnel.png`](/screenshots/02_config_hub_tunnel.png) | Consola de R1 mostrando la interfaz `Tunnel0` con `tunnel mode gre multipoint`, `ip nhrp network-id` y `tunnel protection ipsec profile`. |
| 3 | [`03_config_hub_eigrp_splithorizon.png`](/screenshots/03_config_hub_eigrp_splithorizon.png) | Consola de R1 mostrando `no ip split-horizon eigrp 1` aplicado en `Tunnel0` — paso crítico para que las rutas se propaguen entre Spokes. |
| 4 | [`04_config_spoke1_nhrp.png`](/screenshots/04_config_spoke1_nhrp.png) | Consola de R2 mostrando `ip nhrp nhs`, `ip nhrp map` apuntando al Hub y el resto de la configuración de la interfaz mGRE. |
| 5 | [`05_config_spoke2_completa.png`](/screenshots/05_config_spoke2_completa.png) | Configuración completa de R3 (Spoke 2) mostrando la simetría con R2. |
| 6 | [`06_show_dmvpn_hub.png`](/screenshots/06_show_dmvpn_hub.png) | Salida de `show dmvpn` en R1 mostrando ambos Spokes registrados con estado `UP` y `Attrb: D`. |
| 7 | [`07_isakmp_sa_dos_peers.png`](/screenshots/07_isakmp_sa_dos_peers.png) | Salida de `show crypto isakmp sa` en R1 mostrando dos SAs activas en `QM_IDLE` — una por cada Spoke. |
| 8 | [`08_ping_spoke_a_spoke.png`](/screenshots/08_ping_spoke_a_spoke.png) | Ping exitoso desde PC2 (`20.25.37.66`) hacia PC3 (`20.25.37.130`) — comunicación directa entre Spokes. |
| 9 | [`09_nhrp_dinamico_post_ping.png`](/screenshots/09_nhrp_dinamico_post_ping.png) | Salida de `show ip nhrp` en R2 mostrando la entrada dinámica hacia `10.25.37.3` creada después del ping — evidencia de la resolución NHRP Spoke-to-Spoke. |
| 10 | [`10_eigrp_route_nexthop_directo.png`](/screenshots/10_eigrp_route_nexthop_directo.png) | Salida de `show ip route eigrp` en R2 mostrando la ruta hacia la LAN de Spoke 2 con next-hop directo `10.25.37.3` (no vía el Hub). |
