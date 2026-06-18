# Simulate Offline Locally

Pull images first (requires internet this one time), then start and verify:

```bash
# 1. Pull both images
docker compose pull
docker compose --profile pypiserver pull

# 2. Create .env and start
cat > .env << EOF
PIP_INDEX_URL=http://pypiserver:8080/simple/
PIP_TRUSTED_HOST=pypiserver
EOF
docker compose --profile offline up -d

# 3. Verify xinference can reach pypiserver
docker exec xinference curl -s http://pypiserver:8080/simple/

# 4. Verify host can reach xinference
curl -s http://localhost:9997/v1/models
```

## Block Internet from xinference Container

To simulate the air-gap at the container level while keeping host access:

```bash
# 5. Get container IPs
XINF_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' xinference)
PYP_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' pypiserver)

# 6. Allow established connections (required for host ↔ xinference to work)
sudo iptables -I DOCKER-USER 1 -s $XINF_IP -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# 7. Allow xinference ↔ pypiserver
sudo iptables -I DOCKER-USER 2 -s $XINF_IP -d $PYP_IP -j ACCEPT

# 8. Drop all other outbound from xinference (blocks internet)
sudo iptables -I DOCKER-USER 3 -s $XINF_IP -j DROP

# 9. Verify internet is blocked — should say "OFFLINE"
docker exec xinference curl -s --connect-timeout 3 https://pypi.org \
  && echo "STILL ONLINE" \
  || echo "OFFLINE — good"

# 10. Verify pip is configured to use pypiserver
docker exec xinference python -c "import os; print(os.environ.get('PIP_INDEX_URL'))"
# → http://pypiserver:8080/simple/

# 11. Install a package from pypiserver — should succeed
docker exec xinference pip install --force-reinstall --no-deps --timeout 10 requests

# 12. Try reaching PyPI via pip — should fail (no internet)
docker exec xinference pip install --timeout 5 --index-url https://pypi.org/simple/ pip \
  && echo "STILL ONLINE — something is wrong" \
  || echo "OFFLINE — pip correctly blocked"

# 13. Host access should still work
curl -s http://localhost:9997/v1/models
```

**Clean up rules when done (or restart Docker):**

```bash
sudo iptables -D DOCKER-USER -s $XINF_IP -j DROP
sudo iptables -D DOCKER-USER -s $XINF_IP -d $PYP_IP -j ACCEPT
sudo iptables -D DOCKER-USER -s $XINF_IP -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
```

## How It Works

- `DOCKER-USER` is Docker's chain for user-defined rules — rules here apply to container traffic without interfering with Docker's own networking.
- Rule order: ESTABLISHED first (host access responses), then pypiserver (inter-container), then DROP everything else (internet).
- Host → xinference uses Docker's DNAT port mapping (PREROUTING chain), which happens before the filter rules, so the inbound path is unaffected. The ESTABLISHED rule lets responses back through.
