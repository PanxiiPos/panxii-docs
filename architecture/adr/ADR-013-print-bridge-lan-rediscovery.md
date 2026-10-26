# ADR-013: Tim lai PC in khi DHCP doi IPv4

- Status: Accepted
- Date: 2026-10-26

## Context

Bridge va mobile luu IPv4 luc ghep noi. Router co the cap IPv4 khac sau khi PC
khoi dong lai. Bind vao IP cu lam bridge khong khoi dong; URL cu lam app mat ket noi.
Nguoi dung chon tu tim lai PC thay vi bat buoc dat DHCP reservation.

## Decision

- Giu HTTPS TCP 8765, token, SHA-256 pin va SQLite dedup cua ADR-012.
- Bridge listen IPv4 wildcard de song qua DHCP changes, chi chap nhan request
  HTTP tu RFC1918/loopback. Firewall bat buoc Private/LocalSubnet, khong Public
  hoac port-forward. Discovery UDP 8766 chi tra loi nguon RFC1918.
- Mobile thu URL da luu truoc. Khi loi transport/TLS, broadcast toi LAN trong
  3 giay, gui lai moi 700 ms. Khong quet tung IP, khong qua backend.
- Query JSON `panxii-print-v1` chua nonce ngau nhien 16 byte (hex) va fingerprint,
  khong chua token. Reply chua protocol, nonce, HTTPS port va HMAC-SHA256 hex
  cua `nonce:port:fingerprint`, key la token. Fingerprint duoc lowercase.
  Khong tra token, fingerprint, printer name hay noi dung hoa don.
- Chi nhan reply dung nonce/proof, IP nguon la dia chi ung vien (toi da 8).
  Sau do health check HTTPS phai xac minh pin cu va token cu. Chi luu URL moi
  trong secure storage tenant/user sau khi xac minh; khong tu ghep PC la.
- Retry giu nguyen UUID/payload da persist, khong tao ban in moi. Khong tu
  retry qua PC khac khi token/chung chi thay doi.
- Setup luu InterfaceGuid de show-pairing hien IPv4 hien tai cua dung adapter.
  Khong tu chon VPN/adapter khac neu adapter da chon khong con IPv4.

## Alternatives considered

- DHCP reservation: van khuyen nghi neu router cho phep, nhung khong bat buoc.
- Hostname/mDNS: can them resolver/advertiser va phu thuoc ho tro ten mien LAN.
- Quet subnet: cham, mo rong pham vi truy cap va kho gioi han tren mang lon.

## Consequences

Can nang cap ca app/bridge va mo them UDP 8766 Private/LocalSubnet. Router/client
isolation, VPN hay firewall co the chan broadcast; khi do bao loi ket noi, cho phep
nhap URL hien tai thu cong. Khong bao dam discovery qua subnet/router khac.
Bridge wildcard listen tren moi adapter, nen firewall scope la dieu kien bat buoc.
Android/desktop dung UDP truc tiep. iOS can quyen Local Network va entitlement
`com.apple.developer.networking.multicast` duoc Apple chap thuan/cau hinh trong
signing profile de broadcast; repository chua cau hinh entitlement nay. Khong
co entitlement thi dung URL hien tai hoac DHCP reservation, khong tat TLS pin.

## Related documents

- [ADR-012](ADR-012-windows-wifi-invoice-printing.md)
- [Windows bridge](../../../tools/windows-print-bridge/README.md)
