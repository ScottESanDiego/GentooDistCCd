#FROM gentoo/stage3-amd64
FROM gentoo/stage3:nomultilib-systemd AS toolchain

ARG CLANG_SLOT=22

ENV TARGETCHOST=armv7a-unknown-linux-gnueabihf

EXPOSE 3632

RUN touch /etc/init.d/functions.sh && \
  echo 'EMERGE_DEFAULT_OPTS="--ask=n --jobs=8"' >> /etc/portage/make.conf && \
  echo 'GENTOO_MIRRORS="http://distfiles.gentoo.org"' >> /etc/portage/make.conf && \
  echo 'FEATURES="noman nodoc noinfo"' >> /etc/portage/make.conf

RUN mkdir -p /etc/portage/repos.conf

RUN mkdir -p /var/db/repos/gentoo

RUN ( \
  echo '[gentoo]'  && \
  echo 'location = /var/db/repos/gentoo' && \
  echo 'sync-type = rsync' && \
  echo 'sync-uri = rsync://rsync.us.gentoo.org/gentoo-portage' && \
  echo 'auto-sync = yes' \
  ) > /etc/portage/repos.conf/gentoo.conf

# Setup the portage directory and permissions 
RUN mkdir -p /var/db/repos/gentoo/{distfiles,metadata,packages}
RUN chown -R portage:portage /var/db/repos/gentoo
RUN echo "masters = gentoo" > /var/db/repos/gentoo/metadata/layout.conf

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

RUN ( \
    echo "#!/bin/sh" && \
    echo "export CHOST=${TARGETCHOST}" && \
    echo "eval \"\`gcc-config -E\`\"" && \
    echo 'export PATH="/usr/local/libexec/distcc-clang:$PATH"' && \
    echo "exec distccd \"\$@\"" \
  ) > /usr/local/sbin/distccd-launcher && \
  chmod +x /usr/local/sbin/distccd-launcher

# Sync portage 
RUN emerge-webrsync -q && \
	emerge --sync && \
	eselect news read new && \
	env-update && \
	emerge crossdev && \
	crossdev -t ${TARGETCHOST} --genv 'USE="cxx nls sanitize default-stack-clash-protection default-znow -multilib -fortran -mudflap -openmp"' && \
	mkdir -p /etc/portage/package.use && \
	printf '%s\n' 'llvm-core/llvm llvm_targets_ARM llvm_targets_X86' \
	  'llvm-core/clang llvm_targets_ARM llvm_targets_X86 -extra -static-analyzer' \
	  > /etc/portage/package.use/distcc-clang && \
	emerge distcc "llvm-core/clang:${CLANG_SLOT}"

# Keep the host compiler intact; only distccd sees these ARM-targeted wrappers.
# Use absolute compiler paths to avoid recursing through the wrappers.
RUN mkdir -p /usr/local/libexec/distcc-clang /usr/lib/distcc && \
  for compiler in clang clang++; do \
    test -x "/usr/lib/llvm/${CLANG_SLOT}/bin/${compiler}" && \
    printf '#!/bin/sh\nexec /usr/lib/llvm/%s/bin/%s --no-default-config "$@" --target=%s\n' \
      "${CLANG_SLOT}" "${compiler}" "${TARGETCHOST}" \
      > "/usr/local/libexec/distcc-clang/${compiler}" && \
    chmod +x "/usr/local/libexec/distcc-clang/${compiler}" && \
    for name in "${compiler}" "${compiler}-${CLANG_SLOT}" \
      "${TARGETCHOST}-${compiler}" "${TARGETCHOST}-${compiler}-${CLANG_SLOT}"; do \
      if [ "${name}" != "${compiler}" ]; then \
        ln -s "${compiler}" "/usr/local/libexec/distcc-clang/${name}" || exit 1; \
      fi && \
      ln -sf "/usr/local/libexec/distcc-clang/${compiler}" "/usr/lib/distcc/${name}" \
      || exit 1; \
    done || exit 1; \
  done

RUN /usr/local/libexec/distcc-clang/clang --print-targets && \
	rm -r /var/db/repos/gentoo /var/db/pkg /var/cache

# Copy the cleaned filesystem, excluding the build stage's deleted data layers.
# Retain installed libraries and tools so both dynamic cross-compilers work.
FROM scratch
COPY --from=toolchain / /
ENV PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
ENV TARGETCHOST=armv7a-unknown-linux-gnueabihf
EXPOSE 3632

CMD ["/usr/local/sbin/distccd-launcher", "--allow", "0.0.0.0/0", "--user", "distcc", "--log-level", "notice", "--log-stderr", "--no-detach"]
