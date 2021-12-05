FROM gentoo/stage3:nomultilib

ENV TARGETCHOST aarch64-unknown-linux-gnu

EXPOSE 3632

# Setup options for host (not target)
RUN touch /etc/init.d/functions.sh && \
  echo 'EMERGE_DEFAULT_OPTS="--ask=n --jobs=8"' >> /etc/portage/make.conf && \
  echo 'GENTOO_MIRRORS="http://distfiles.gentoo.org"' >> /etc/portage/make.conf && \
  echo 'FEATURES="noman nodoc noinfo -sandbox -usersandbox -ipc-sandbox -network-sandbox -pid-sandbox"' >> /etc/portage/make.conf

RUN mkdir -p /etc/portage/repos.conf

RUN ( \
  echo '[gentoo]'  && \
  echo 'sync-type = webrsync' \
  ) > /etc/portage/repos.conf/gentoo.conf

RUN mkdir -p /usr/local/portage-crossdev/{profiles,metadata} && \
  echo 'crossdev' > /usr/local/portage-crossdev/profiles/repo_name && \
  echo 'masters = gentoo' > /usr/local/portage-crossdev/metadata/layout.conf && \
  chown -R portage:portage /usr/local/portage-crossdev

RUN ( \
    echo "[crossdev]" && \
    echo "location = /usr/local/portage-crossdev" && \
    echo "priority = 10" && \
    echo "masters = gentoo" && \
    echo "auto-sync = no" \
  ) > /etc/portage/repos.conf/crossdev.conf

# Create bash script to launch distccd (our CMD)
RUN ( \
    echo "#!/bin/sh" && \
    echo "export CHOST=${TARGETCHOST}" && \
    echo "eval \"\`gcc-config -E\`\"" && \
    echo "exec distccd \"\$@\"" \
  ) > /usr/local/sbin/distccd-launcher && \
  chmod +x /usr/local/sbin/distccd-launcher

RUN mkdir -p /var/db/repos/gentoo/metadata
RUN echo "masters = gentoo" > /var/db/repos/gentoo/metadata/layout.conf

# Sync portage and emerge crossdev stuff. This is the meat. 
RUN emerge --sync && \
	eselect news read --quiet new && \
	env-update  && \
	emerge crossdev && \
	crossdev -t ${TARGETCHOST} --genv 'USE="cxx nls sanitize -multilib -fortran -mudflap -openmp"' && \
	emerge distcc  && \
	emerge --depclean && \
	rm -r /var/db/repos/gentoo /var/db/pkg /var/cache

CMD ["/usr/local/sbin/distccd-launcher", "--allow", "0.0.0.0/0", "--user", "distcc", "--log-level", "notice", "--log-stderr", "--no-detach"]
