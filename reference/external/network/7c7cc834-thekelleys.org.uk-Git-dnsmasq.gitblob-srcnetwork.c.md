---
title: thekelleys.org.uk Git - dnsmasq.git/blob - src/network.c
source: http://thekelleys.org.uk/gitweb/?p=dnsmasq.git;a=blob;f=src/network.c;hb=1682d15a744880b0398af75eadf68fe66128af78
kind: external
domain: network
original_date: 2007-06-29
fetched_at: 2026-05-16
tags: [external, network]
---

> [!info] 外部文章 · 自动导入
> 来源：[thekelleys.org.uk](http://thekelleys.org.uk/gitweb/?p=dnsmasq.git;a=blob;f=src/network.c;hb=1682d15a744880b0398af75eadf68fe66128af78)
> 原始日期：2007-06-29
> 抓取日期：2026-05-16

# thekelleys.org.uk Git - dnsmasq.git/blob - src/network.c

1 /* dnsmasq is Copyright (c) 2000-2018 Simon Kelley

3 This program is free software; you can redistribute it and/or modify

4 it under the terms of the GNU General Public License as published by

5 the Free Software Foundation; version 2 dated June, 1991, or

6 (at your option) version 3 dated 29 June, 2007.

8 This program is distributed in the hope that it will be useful,

9 but WITHOUT ANY WARRANTY; without even the implied warranty of

10 MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the

11 GNU General Public License for more details.

13 You should have received a copy of the GNU General Public License

14 along with this program. If not, see <http://www.gnu.org/licenses/>.

15 */

17 #include "dnsmasq.h"

19 #ifdef HAVE_LINUX_NETWORK

21 int indextoname(int fd, int index, char *name)

22 {

23 struct ifreq ifr;

25 if (index == 0)

26 return 0;

28 ifr.ifr_ifindex = index;

29 if (ioctl(fd, SIOCGIFNAME, &ifr) == -1)

30 return 0;

32 strncpy(name, ifr.ifr_name, IF_NAMESIZE);

34 return 1;

35 }

38 #elif defined(HAVE_SOLARIS_NETWORK)

40 #include <zone.h>

41 #include <alloca.h>

42 #ifndef LIFC_UNDER_IPMP

43 # define LIFC_UNDER_IPMP 0

44 #endif

46 int indextoname(int fd, int index, char *name)

47 {

48 int64_t lifc_flags;

49 struct lifnum lifn;

50 int numifs, bufsize, i;

51 struct lifconf lifc;

52 struct lifreq *lifrp;

54 if (index == 0)

55 return 0;

57 if (getzoneid() == GLOBAL_ZONEID)

58 {

59 if (!if_indextoname(index, name))

60 return 0;

61 return 1;

62 }

64 lifc_flags = LIFC_NOXMIT | LIFC_TEMPORARY | LIFC_ALLZONES | LIFC_UNDER_IPMP;

65 lifn.lifn_family = AF_UNSPEC;

66 lifn.lifn_flags = lifc_flags;

67 if (ioctl(fd, SIOCGLIFNUM, &lifn) < 0)

68 return 0;

70 numifs = lifn.lifn_count;

71 bufsize = numifs * sizeof(struct lifreq);

73 lifc.lifc_family = AF_UNSPEC;

74 lifc.lifc_flags = lifc_flags;

75 lifc.lifc_len = bufsize;

76 lifc.lifc_buf = alloca(bufsize);

78 if (ioctl(fd, SIOCGLIFCONF, &lifc) < 0)

79 return 0;

81 lifrp = lifc.lifc_req;

82 for (i = lifc.lifc_len / sizeof(struct lifreq); i; i--, lifrp++)

83 {

84 struct lifreq lifr;

85 strncpy(lifr.lifr_name, lifrp->lifr_name, IF_NAMESIZE);

86 if (ioctl(fd, SIOCGLIFINDEX, &lifr) < 0)

87 return 0;

89 if (lifr.lifr_index == index) {

90 strncpy(name, lifr.lifr_name, IF_NAMESIZE);

91 return 1;

92 }

93 }

94 return 0;

95 }

98 #else

100 int indextoname(int fd, int index, char *name)

101 {

102 (void)fd;

104 if (index == 0 || !if_indextoname(index, name))

105 return 0;

107 return 1;

108 }

110 #endif

112 int iface_check(int family, struct all_addr *addr, char *name, int *auth)

113 {

114 struct iname *tmp;

115 int ret = 1, match_addr = 0;

117 /* Note: have to check all and not bail out early, so that we set the

118 "used" flags.

120 May be called with family == AF_LOCALto check interface by name only. */

122 if (auth)

123 *auth = 0;

125 if (daemon->if_names || daemon->if_addrs)

126 {

127 ret = 0;

129 for (tmp = daemon->if_names; tmp; tmp = tmp->next)

130 if (tmp->name && wildcard_match(tmp->name, name))

131 ret = tmp->used = 1;

133 if (addr)

134 for (tmp = daemon->if_addrs; tmp; tmp = tmp->next)

135 if (tmp->addr.sa.sa_family == family)

136 {

137 if (family == AF_INET &&

138 tmp->addr.in.sin_addr.s_addr == addr->addr.addr4.s_addr)

139 ret = match_addr = tmp->used = 1;

140 #ifdef HAVE_IPV6

141 else if (family == AF_INET6 &&

142 IN6_ARE_ADDR_EQUAL(&tmp->addr.in6.sin6_addr,

143 &addr->addr.addr6))

144 ret = match_addr = tmp->used = 1;

145 #endif

146 }

147 }

149 if (!match_addr)

150 for (tmp = daemon->if_except; tmp; tmp = tmp->next)

151 if (tmp->name && wildcard_match(tmp->name, name))

152 ret = 0;

155 for (tmp = daemon->authinterface; tmp; tmp = tmp->next)

156 if (tmp->name)

157 {

158 if (strcmp(tmp->name, name) == 0 &&

159 (tmp->addr.sa.sa_family == 0 || tmp->addr.sa.sa_family == family))

160 break;

161 }

162 else if (addr && tmp->addr.sa.sa_family == AF_INET && family == AF_INET &&

163 tmp->addr.in.sin_addr.s_addr == addr->addr.addr4.s_addr)

164 break;

165 #ifdef HAVE_IPV6

166 else if (addr && tmp->addr.sa.sa_family == AF_INET6 && family == AF_INET6 &&

167 IN6_ARE_ADDR_EQUAL(&tmp->addr.in6.sin6_addr, &addr->addr.addr6))

168 break;

169 #endif

171 if (tmp && auth)

172 {

173 *auth = 1;

174 ret = 1;

175 }

177 return ret;

178 }

181 /* Fix for problem that the kernel sometimes reports the loopback interface as the

182 arrival interface when a packet originates locally, even when sent to address of

183 an interface other than the loopback. Accept packet if it arrived via a loopback

184 interface, even when we're not accepting packets that way, as long as the destination

185 address is one we're believing. Interface list must be up-to-date before calling. */

186 int loopback_exception(int fd, int family, struct all_addr *addr, char *name)

187 {

188 struct ifreq ifr;

189 struct irec *iface;

191 strncpy(ifr.ifr_name, name, IF_NAMESIZE);

192 if (ioctl(fd, SIOCGIFFLAGS, &ifr) != -1 &&

193 ifr.ifr_flags & IFF_LOOPBACK)

194 {

195 for (iface = daemon->interfaces; iface; iface = iface->next)

196 if (iface->addr.sa.sa_family == family)

197 {

198 if (family == AF_INET)

199 {

200 if (iface->addr.in.sin_addr.s_addr == addr->addr.addr4.s_addr)

201 return 1;

202 }

203 #ifdef HAVE_IPV6

204 else if (IN6_ARE_ADDR_EQUAL(&iface->addr.in6.sin6_addr, &addr->addr.addr6))

205 return 1;

206 #endif

208 }

209 }

210 return 0;

211 }

213 /* If we're configured with something like --interface=eth0:0 then we'll listen correctly

214 on the relevant address, but the name of the arrival interface, derived from the

215 index won't match the config. Check that we found an interface address for the arrival

216 interface: daemon->interfaces must be up-to-date. */

217 int label_exception(int index, int family, struct all_addr *addr)

218 {

219 struct irec *iface;

221 /* labels only supported on IPv4 addresses. */

222 if (family != AF_INET)

223 return 0;

225 for (iface = daemon->interfaces; iface; iface = iface->next)

226 if (iface->index == index && iface->addr.sa.sa_family == AF_INET &&

227 iface->addr.in.sin_addr.s_addr == addr->addr.addr4.s_addr)

228 return 1;

230 return 0;

231 }

233 struct iface_param {

234 struct addrlist *spare;

235 int fd;

236 };

238 static int iface_allowed(struct iface_param *param, int if_index, char *label,

239 union mysockaddr *addr, struct in_addr netmask, int prefixlen, int iface_flags)

240 {

241 struct irec *iface;

242 int mtu = 0, loopback;

243 struct ifreq ifr;

244 int tftp_ok = !!option_bool(OPT_TFTP);

245 int dhcp_ok = 1;

246 int auth_dns = 0;

247 int is_label = 0;

248 #if defined(HAVE_DHCP) || defined(HAVE_TFTP)

249 struct iname *tmp;

250 #endif

252 (void)prefixlen;

254 if (!indextoname(param->fd, if_index, ifr.ifr_name) ||

255 ioctl(param->fd, SIOCGIFFLAGS, &ifr) == -1)

256 return 0;

258 loopback = ifr.ifr_flags & IFF_LOOPBACK;

260 if (loopback)

261 dhcp_ok = 0;

263 if (ioctl(param->fd, SIOCGIFMTU, &ifr) != -1)

264 mtu = ifr.ifr_mtu;

266 if (!label)

267 label = ifr.ifr_name;

268 else

269 is_label = strcmp(label, ifr.ifr_name);

271 /* maintain a list of all addresses on all interfaces for --local-service option */

272 if (option_bool(OPT_LOCAL_SERVICE))

273 {

274 struct addrlist *al;

276 if (param->spare)

277 {

278 al = param->spare;

279 param->spare = al->next;

280 }

281 else

282 al = whine_malloc(sizeof(struct addrlist));

284 if (al)

285 {

286 al->next = daemon->interface_addrs;

287 daemon->interface_addrs = al;

288 al->prefixlen = prefixlen;

290 if (addr->sa.sa_family == AF_INET)

291 {

292 al->addr.addr.addr4 = addr->in.sin_addr;

293 al->flags = 0;

294 }

295 #ifdef HAVE_IPV6

296 else

297 {

298 al->addr.addr.addr6 = addr->in6.sin6_addr;

299 al->flags = ADDRLIST_IPV6;

300 }

301 #endif

302 }

303 }

305 #ifdef HAVE_IPV6

306 if (addr->sa.sa_family != AF_INET6 || !IN6_IS_ADDR_LINKLOCAL(&addr->in6.sin6_addr))

307 #endif

308 {

309 struct interface_name *int_name;

310 struct addrlist *al;

311 #ifdef HAVE_AUTH

312 struct auth_zone *zone;

313 struct auth_name_list *name;

315 /* Find subnets in auth_zones */

316 for (zone = daemon->auth_zones; zone; zone = zone->next)

317 for (name = zone->interface_names; name; name = name->next)

318 if (wildcard_match(name->name, label))

319 {

320 if (addr->sa.sa_family == AF_INET && (name->flags & AUTH4))

321 {

322 if (param->spare)

323 {

324 al = param->spare;

325 param->spare = al->next;

326 }

327 else

328 al = whine_malloc(sizeof(struct addrlist));

330 if (al)

331 {

332 al->next = zone->subnet;

333 zone->subnet = al;

334 al->prefixlen = prefixlen;

335 al->addr.addr.addr4 = addr->in.sin_addr;

336 al->flags = 0;

337 }

338 }

340 #ifdef HAVE_IPV6

341 if (addr->sa.sa_family == AF_INET6 && (name->flags & AUTH6))

342 {

343 if (param->spare)

344 {

345 al = param->spare;

346 param->spare = al->next;

347 }

348 else

349 al = whine_malloc(sizeof(struct addrlist));

351 if (al)

352 {

353 al->next = zone->subnet;

354 zone->subnet = al;

355 al->prefixlen = prefixlen;

356 al->addr.addr.addr6 = addr->in6.sin6_addr;

357 al->flags = ADDRLIST_IPV6;

358 }

359 }

360 #endif

362 }

363 #endif

365 /* Update addresses from interface_names. These are a set independent

366 of the set we're listening on. */

367 for (int_name = daemon->int_names; int_name; int_name = int_name->next)

368 if (strncmp(label, int_name->intr, IF_NAMESIZE) == 0 &&

369 (addr->sa.sa_family == int_name->family || int_name->family == 0))

370 {

371 if (param->spare)

372 {

373 al = param->spare;

374 param->spare = al->next;

375 }

376 else

377 al = whine_malloc(sizeof(struct addrlist));

379 if (al)

380 {

381 al->next = int_name->addr;

382 int_name->addr = al;

384 if (addr->sa.sa_family == AF_INET)

385 {

386 al->addr.addr.addr4 = addr->in.sin_addr;

387 al->flags = 0;

388 }

389 #ifdef HAVE_IPV6

390 else

391 {

392 al->addr.addr.addr6 = addr->in6.sin6_addr;

393 al->flags = ADDRLIST_IPV6;

394 /* Privacy addresses and addresses still undergoing DAD and deprecated addresses

395 don't appear in forward queries, but will in reverse ones. */

396 if (!(iface_flags & IFACE_PERMANENT) || (iface_flags & (IFACE_DEPRECATED | IFACE_TENTATIVE)))

397 al->flags |= ADDRLIST_REVONLY;

398 }

399 #endif

400 }

401 }

402 }

404 /* check whether the interface IP has been added already

405 we call this routine multiple times. */

406 for (iface = daemon->interfaces; iface; iface = iface->next)

407 if (sockaddr_isequal(&iface->addr, addr))

408 {

409 iface->dad = !!(iface_flags & IFACE_TENTATIVE);

410 iface->found = 1; /* for garbage collection */

411 return 1;

412 }

414 /* If we are restricting the set of interfaces to use, make

415 sure that loopback interfaces are in that set. */

416 if (daemon->if_names && loopback)

417 {

418 struct iname *lo;

419 for (lo = daemon->if_names; lo; lo = lo->next)

420 if (lo->name && strcmp(lo->name, ifr.ifr_name) == 0)

421 break;

423 if (!lo && (lo = whine_malloc(sizeof(struct iname))))

424 {

425 if ((lo->name = whine_malloc(strlen(ifr.ifr_name)+1)))

426 {

427 strcpy(lo->name, ifr.ifr_name);

428 lo->used = 1;

429 lo->next = daemon->if_names;

430 daemon->if_names = lo;

431 }

432 else

433 free(lo);

434 }

435 }

437 if (addr->sa.sa_family == AF_INET &&

438 !iface_check(AF_INET, (struct all_addr *)&addr->in.sin_addr, label, &auth_dns))

439 return 1;

441 #ifdef HAVE_IPV6

442 if (addr->sa.sa_family == AF_INET6 &&

443 !iface_check(AF_INET6, (struct all_addr *)&addr->in6.sin6_addr, label, &auth_dns))

444 return 1;

445 #endif

447 #ifdef HAVE_DHCP

448 /* No DHCP where we're doing auth DNS. */

449 if (auth_dns)

450 {

451 tftp_ok = 0;

452 dhcp_ok = 0;

453 }

454 else

455 for (tmp = daemon->dhcp_except; tmp; tmp = tmp->next)

456 if (tmp->name && wildcard_match(tmp->name, ifr.ifr_name))

457 {

458 tftp_ok = 0;

459 dhcp_ok = 0;

460 }

461 #endif

464 #ifdef HAVE_TFTP

465 if (daemon->tftp_interfaces)

466 {

467 /* dedicated tftp interface list */

468 tftp_ok = 0;

469 for (tmp = daemon->tftp_interfaces; tmp; tmp = tmp->next)

470 if (tmp->name && wildcard_match(tmp->name, ifr.ifr_name))

471 tftp_ok = 1;

472 }

473 #endif

475 /* add to list */

476 if ((iface = whine_malloc(sizeof(struct irec))))

477 {

478 iface->addr = *addr;

479 iface->netmask = netmask;

480 iface->tftp_ok = tftp_ok;

481 iface->dhcp_ok = dhcp_ok;

482 iface->dns_auth = auth_dns;

483 iface->mtu = mtu;

484 iface->dad = !!(iface_flags & IFACE_TENTATIVE);

485 iface->found = 1;

486 iface->done = iface->multicast_done = iface->warned = 0;

487 iface->index = if_index;

488 iface->label = is_label;

489 if ((iface->name = whine_malloc(strlen(ifr.ifr_name)+1)))

490 {

491 strcpy(iface->name, ifr.ifr_name);

492 iface->next = daemon->interfaces;

493 daemon->interfaces = iface;

494 return 1;

495 }

496 free(iface);

498 }

500 errno = ENOMEM;

501 return 0;

502 }

504 #ifdef HAVE_IPV6

505 static int iface_allowed_v6(struct in6_addr *local, int prefix,

506 int scope, int if_index, int flags,

507 int preferred, int valid, void *vparam)

508 {

509 union mysockaddr addr;

510 struct in_addr netmask; /* dummy */

511 netmask.s_addr = 0;

513 (void)scope; /* warning */

514 (void)preferred;

515 (void)valid;

517 memset(&addr, 0, sizeof(addr));

518 #ifdef HAVE_SOCKADDR_SA_LEN

519 addr.in6.sin6_len = sizeof(addr.in6);

520 #endif

521 addr.in6.sin6_family = AF_INET6;

522 addr.in6.sin6_addr = *local;

523 addr.in6.sin6_port = htons(daemon->port);

524 /* FreeBSD insists this is zero for non-linklocal addresses */

525 if (IN6_IS_ADDR_LINKLOCAL(local))

526 addr.in6.sin6_scope_id = if_index;

527 else

528 addr.in6.sin6_scope_id = 0;

530 return iface_allowed((struct iface_param *)vparam, if_index, NULL, &addr, netmask, prefix, flags);

531 }

532 #endif

534 static int iface_allowed_v4(struct in_addr local, int if_index, char *label,

535 struct in_addr netmask, struct in_addr broadcast, void *vparam)

536 {

537 union mysockaddr addr;

538 int prefix, bit;

540 (void)broadcast; /* warning */

542 memset(&addr, 0, sizeof(addr));

543 #ifdef HAVE_SOCKADDR_SA_LEN

544 addr.in.sin_len = sizeof(addr.in);

545 #endif

546 addr.in.sin_family = AF_INET;

547 addr.in.sin_addr = local;

548 addr.in.sin_port = htons(daemon->port);

550 /* determine prefix length from netmask */

551 for (prefix = 32, bit = 1; (bit & ntohl(netmask.s_addr)) == 0 && prefix != 0; bit = bit << 1, prefix--);

553 return iface_allowed((struct iface_param *)vparam, if_index, label, &addr, netmask, prefix, 0);

554 }

556 int enumerate_interfaces(int reset)

557 {

558 static struct addrlist *spare = NULL;

559 static int done = 0;

560 struct iface_param param;

561 int errsave, ret = 1;

562 struct addrlist *addr, *tmp;

563 struct interface_name *intname;

564 struct irec *iface;

565 #ifdef HAVE_AUTH

566 struct auth_zone *zone;

567 #endif

569 /* Do this max once per select cycle - also inhibits netlink socket use

570 in TCP child processes. */

572 if (reset)

573 {

574 done = 0;

575 return 1;

576 }

578 if (done)

579 return 1;

581 done = 1;

583 if ((param.fd = socket(PF_INET, SOCK_DGRAM, 0)) == -1)

584 return 0;

586 /* Mark interfaces for garbage collection */

587 for (iface = daemon->interfaces; iface; iface = iface->next)

588 iface->found = 0;

590 /* remove addresses stored against interface_names */

591 for (intname = daemon->int_names; intname; intname = intname->next)

592 {

593 for (addr = intname->addr; addr; addr = tmp)

594 {

595 tmp = addr->next;

596 addr->next = spare;

597 spare = addr;

598 }

600 intname->addr = NULL;

601 }

603 /* Remove list of addresses of local interfaces */

604 for (addr = daemon->interface_addrs; addr; addr = tmp)

605 {

606 tmp = addr->next;

607 addr->next = spare;

608 spare = addr;

609 }

610 daemon->interface_addrs = NULL;

612 #ifdef HAVE_AUTH

613 /* remove addresses stored against auth_zone subnets, but not

614 ones configured as address literals */

615 for (zone = daemon->auth_zones; zone; zone = zone->next)

616 if (zone->interface_names)

617 {

618 struct addrlist **up;

619 for (up = &zone->subnet, addr = zone->subnet; addr; addr = tmp)

620 {

621 tmp = addr->next;

622 if (addr->flags & ADDRLIST_LITERAL)

623 up = &addr->next;

624 else

625 {

626 *up = addr->next;

627 addr->next = spare;

628 spare = addr;

629 }

630 }

631 }

632 #endif

634 param.spare = spare;

636 #ifdef HAVE_IPV6

637 ret = iface_enumerate(AF_INET6, ¶m, iface_allowed_v6);

638 #endif

640 if (ret)

641 ret = iface_enumerate(AF_INET, ¶m, iface_allowed_v4);

643 errsave = errno;

644 close(param.fd);

646 if (option_bool(OPT_CLEVERBIND))

647 {

648 /* Garbage-collect listeners listening on addresses that no longer exist.

649 Does nothing when not binding interfaces or for listeners on localhost,

650 since the ->iface field is NULL. Note that this needs the protections

651 against reentrancy, hence it's here. It also means there's a possibility,

652 in OPT_CLEVERBIND mode, that at listener will just disappear after

653 a call to enumerate_interfaces, this is checked OK on all calls. */

654 struct listener *l, *tmp, **up;

656 for (up = &daemon->listeners, l = daemon->listeners; l; l = tmp)

657 {

658 tmp = l->next;

660 if (!l->iface || l->iface->found)

661 up = &l->next;

662 else

663 {

664 *up = l->next;

666 /* In case it ever returns */

667 l->iface->done = 0;

669 if (l->fd != -1)

670 close(l->fd);

671 if (l->tcpfd != -1)

672 close(l->tcpfd);

673 if (l->tftpfd != -1)

674 close(l->tftpfd);

676 free(l);

677 }

678 }

679 }

681 errno = errsave;

682 spare = param.spare;

684 return ret;

685 }

687 /* set NONBLOCK bit on fd: See Stevens 16.6 */

688 int fix_fd(int fd)

689 {

690 int flags;

692 if ((flags = fcntl(fd, F_GETFL)) == -1 ||

693 fcntl(fd, F_SETFL, flags | O_NONBLOCK) == -1)

694 return 0;

696 return 1;

697 }

699 static int make_sock(union mysockaddr *addr, int type, int dienow)

700 {

701 int family = addr->sa.sa_family;

702 int fd, rc, opt = 1;

704 if ((fd = socket(family, type, 0)) == -1)

705 {

706 int port, errsave;

707 char *s;

709 /* No error if the kernel just doesn't support this IP flavour */

710 if (errno == EPROTONOSUPPORT ||

711 errno == EAFNOSUPPORT ||

712 errno == EINVAL)

713 return -1;

715 err:

716 errsave = errno;

717 port = prettyprint_addr(addr, daemon->addrbuff);

718 if (!option_bool(OPT_NOWILD) && !option_bool(OPT_CLEVERBIND))

719 sprintf(daemon->addrbuff, "port %d", port);

720 s = _("failed to create listening socket for %s: %s");

722 if (fd != -1)

723 close (fd);

725 errno = errsave;

727 if (dienow)

728 {

729 /* failure to bind addresses given by --listen-address at this point

730 is OK if we're doing bind-dynamic */

731 if (!option_bool(OPT_CLEVERBIND))

732 die(s, daemon->addrbuff, EC_BADNET);

733 }

734 else

735 my_syslog(LOG_WARNING, s, daemon->addrbuff, strerror(errno));

737 return -1;

738 }

740 if (setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt)) == -1 || !fix_fd(fd))

741 goto err;

743 #ifdef HAVE_IPV6

744 if (family == AF_INET6 && setsockopt(fd, IPPROTO_IPV6, IPV6_V6ONLY, &opt, sizeof(opt)) == -1)

745 goto err;

746 #endif

748 if ((rc = bind(fd, (struct sockaddr *)addr, sa_len(addr))) == -1)

749 goto err;

751 if (type == SOCK_STREAM)

752 {

753 if (listen(fd, TCP_BACKLOG) == -1)

754 goto err;

755 }

756 else if (family == AF_INET)

757 {

758 if (!option_bool(OPT_NOWILD))

759 {

760 #if defined(HAVE_LINUX_NETWORK)

761 if (setsockopt(fd, IPPROTO_IP, IP_PKTINFO, &opt, sizeof(opt)) == -1)

762 goto err;

763 #elif defined(IP_RECVDSTADDR) && defined(IP_RECVIF)

764 if (setsockopt(fd, IPPROTO_IP, IP_RECVDSTADDR, &opt, sizeof(opt)) == -1 ||

765 setsockopt(fd, IPPROTO_IP, IP_RECVIF, &opt, sizeof(opt)) == -1)

766 goto err;

767 #endif

768 }

769 }

770 #ifdef HAVE_IPV6

771 else if (!set_ipv6pktinfo(fd))

772 goto err;

773 #endif

775 return fd;

776 }

778 #ifdef HAVE_IPV6

779 int set_ipv6pktinfo(int fd)

780 {

781 int opt = 1;

783 /* The API changed around Linux 2.6.14 but the old ABI is still supported:

784 handle all combinations of headers and kernel.

785 OpenWrt note that this fixes the problem addressed by your very broken patch. */

786 daemon->v6pktinfo = IPV6_PKTINFO;

788 #ifdef IPV6_RECVPKTINFO

789 if (setsockopt(fd, IPPROTO_IPV6, IPV6_RECVPKTINFO, &opt, sizeof(opt)) != -1)

790 return 1;

791 # ifdef IPV6_2292PKTINFO

792 else if (errno == ENOPROTOOPT && setsockopt(fd, IPPROTO_IPV6, IPV6_2292PKTINFO, &opt, sizeof(opt)) != -1)

793 {

794 daemon->v6pktinfo = IPV6_2292PKTINFO;

795 return 1;

796 }

797 # endif

798 #else

799 if (setsockopt(fd, IPPROTO_IPV6, IPV6_PKTINFO, &opt, sizeof(opt)) != -1)

800 return 1;

801 #endif

803 return 0;

804 }

805 #endif

808 /* Find the interface on which a TCP connection arrived, if possible, or zero otherwise. */

809 int tcp_interface(int fd, int af)

810 {

811 int if_index = 0;

813 #ifdef HAVE_LINUX_NETWORK

814 int opt = 1;

815 struct cmsghdr *cmptr;

816 struct msghdr msg;

817 socklen_t len;

819 /* use mshdr so that the CMSDG_* macros are available */

820 msg.msg_control = daemon->packet;

821 msg.msg_controllen = len = daemon->packet_buff_sz;

823 /* we overwrote the buffer... */

824 daemon->srv_save = NULL;

826 if (af == AF_INET)

827 {

828 if (setsockopt(fd, IPPROTO_IP, IP_PKTINFO, &opt, sizeof(opt)) != -1 &&

829 getsockopt(fd, IPPROTO_IP, IP_PKTOPTIONS, msg.msg_control, &len) != -1)

830 {

831 msg.msg_controllen = len;

832 for (cmptr = CMSG_FIRSTHDR(&msg); cmptr; cmptr = CMSG_NXTHDR(&msg, cmptr))

833 if (cmptr->cmsg_level == IPPROTO_IP && cmptr->cmsg_type == IP_PKTINFO)

834 {

835 union {

836 unsigned char *c;

837 struct in_pktinfo *p;

838 } p;

840 p.c = CMSG_DATA(cmptr);

841 if_index = p.p->ipi_ifindex;

842 }

843 }

844 }

845 #ifdef HAVE_IPV6

846 else

847 {

848 /* Only the RFC-2292 API has the ability to find the interface for TCP connections,

849 it was removed in RFC-3542 !!!!

851 Fortunately, Linux kept the 2292 ABI when it moved to 3542. The following code always

852 uses the old ABI, and should work with pre- and post-3542 kernel headers */

854 #ifdef IPV6_2292PKTOPTIONS

855 # define PKTOPTIONS IPV6_2292PKTOPTIONS

856 #else

857 # define PKTOPTIONS IPV6_PKTOPTIONS

858 #endif

860 if (set_ipv6pktinfo(fd) &&

861 getsockopt(fd, IPPROTO_IPV6, PKTOPTIONS, msg.msg_control, &len) != -1)

862 {

863 msg.msg_controllen = len;

864 for (cmptr = CMSG_FIRSTHDR(&msg); cmptr; cmptr = CMSG_NXTHDR(&msg, cmptr))

865 if (cmptr->cmsg_level == IPPROTO_IPV6 && cmptr->cmsg_type == daemon->v6pktinfo)

866 {

867 union {

868 unsigned char *c;

869 struct in6_pktinfo *p;

870 } p;

871 p.c = CMSG_DATA(cmptr);

873 if_index = p.p->ipi6_ifindex;

874 }

875 }

876 }

877 #endif /* IPV6 */

878 #endif /* Linux */

880 return if_index;

881 }

883 static struct listener *create_listeners(union mysockaddr *addr, int do_tftp, int dienow)

884 {

885 struct listener *l = NULL;

886 int fd = -1, tcpfd = -1, tftpfd = -1;

888 (void)do_tftp;

890 if (daemon->port != 0)

891 {

892 fd = make_sock(addr, SOCK_DGRAM, dienow);

893 tcpfd = make_sock(addr, SOCK_STREAM, dienow);

894 }

896 #ifdef HAVE_TFTP

897 if (do_tftp)

898 {

899 if (addr->sa.sa_family == AF_INET)

900 {

901 /* port must be restored to DNS port for TCP code */

902 short save = addr->in.sin_port;

903 addr->in.sin_port = htons(TFTP_PORT);

904 tftpfd = make_sock(addr, SOCK_DGRAM, dienow);

905 addr->in.sin_port = save;

906 }

907 # ifdef HAVE_IPV6

908 else

909 {

910 short save = addr->in6.sin6_port;

911 addr->in6.sin6_port = htons(TFTP_PORT);

912 tftpfd = make_sock(addr, SOCK_DGRAM, dienow);

913 addr->in6.sin6_port = save;

914 }

915 # endif

916 }

917 #endif

919 if (fd != -1 || tcpfd != -1 || tftpfd != -1)

920 {

921 l = safe_malloc(sizeof(struct listener));

922 l->next = NULL;

923 l->family = addr->sa.sa_family;

924 l->fd = fd;

925 l->tcpfd = tcpfd;

926 l->tftpfd = tftpfd;

927 l->iface = NULL;

928 }

930 return l;

931 }

933 void create_wildcard_listeners(void)

934 {

935 union mysockaddr addr;

936 struct listener *l, *l6;

938 memset(&addr, 0, sizeof(addr));

939 #ifdef HAVE_SOCKADDR_SA_LEN

940 addr.in.sin_len = sizeof(addr.in);

941 #endif

942 addr.in.sin_family = AF_INET;

943 addr.in.sin_addr.s_addr = INADDR_ANY;

944 addr.in.sin_port = htons(daemon->port);

946 l = create_listeners(&addr, !!option_bool(OPT_TFTP), 1);

948 #ifdef HAVE_IPV6

949 memset(&addr, 0, sizeof(addr));

950 # ifdef HAVE_SOCKADDR_SA_LEN

951 addr.in6.sin6_len = sizeof(addr.in6);

952 # endif

953 addr.in6.sin6_family = AF_INET6;

954 addr.in6.sin6_addr = in6addr_any;

955 addr.in6.sin6_port = htons(daemon->port);

957 l6 = create_listeners(&addr, !!option_bool(OPT_TFTP), 1);

958 if (l)

959 l->next = l6;

960 else

961 l 

[... 内容超长，已截断；完整原文见 source URL ...]
