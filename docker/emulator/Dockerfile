ARG DOCKER_ANDROID_VERSION=v3.9.0-p2
FROM budtmo/docker-android:base_${DOCKER_ANDROID_VERSION}

#==================
# Android Packages
#==================
ARG EMULATOR_ANDROID_VERSION
ARG EMULATOR_API_LEVEL
ENV EMULATOR_ANDROID_VERSION=${EMULATOR_ANDROID_VERSION} \
    EMULATOR_API_LEVEL=${EMULATOR_API_LEVEL} \
    EMULATOR_SYS_IMG=x86_64 \
    EMULATOR_IMG_TYPE=google_apis \
    EMULATOR_BROWSER=chrome
ENV PATH=${PATH}:${ANDROID_HOME}/build-tools
RUN yes | sdkmanager --licenses \
 && sdkmanager "platforms;android-${EMULATOR_API_LEVEL}" \
 "system-images;android-${EMULATOR_API_LEVEL};${EMULATOR_IMG_TYPE};${EMULATOR_SYS_IMG}" "emulator" \
 && ln -s ${ANDROID_HOME}/emulator/emulator /usr/bin/

#=============
# UI Packages
#-------------
# feh       Screen background
# x11vnc    VNC server
# xterm     Terminal emulator
# xvfb      Virtual display (NepStack: was installed by upstream's private build step)
# openbox   Window manager (NepStack: same)
# sudo      Used by the CLI to grant KVM access inside the container
#==================
RUN apt-get -qqy update && apt-get -qqy install --no-install-recommends \
    feh \
    x11vnc \
    xterm \
    xvfb \
    openbox \
    sudo \
 && apt autoremove -y \
 && apt clean all \
 && rm -rf /var/lib/apt/lists/*

#=====================================================
# Emulator runtime libraries
# (NepStack: were installed by upstream's private build step)
#=====================================================
RUN apt-get -qqy update \
 && for p in libpulse0 libnss3 libgl1 libegl1 libgbm1 libdrm2 libdbus-1-3 libbsd0 libx11-6 libx11-xcb1 \
      libxcomposite1 libxcursor1 libxdamage1 libxext6 libxfixes3 libxi6 libxkbfile1 libxrandr2 libxrender1 \
      libxtst6 libxkbcommon0 libxkbcommon-x11-0 libxcb1 libxcb-cursor0 libxcb-icccm4 libxcb-image0 \
      libxcb-keysyms1 libxcb-randr0 libxcb-render-util0 libxcb-shape0 libxcb-xfixes0 libxcb-xinerama0 \
      libxcb-xkb1 libfontconfig1 libfreetype6 libasound2t64 libasound2; do \
      apt-get -qqy install --no-install-recommends "$p" >/dev/null 2>&1 && echo "installed $p" || echo "skipped $p"; \
    done \
 && apt clean all \
 && rm -rf /var/lib/apt/lists/* \
 && echo "Emulator library check:" \
 && (ldd ${ANDROID_HOME}/emulator/qemu/linux-x86_64/qemu-system-x86_64 | grep 'not found' || echo "all libraries found")

#=======
# noVNC
#=======
ENV NOVNC_VERSION="1.7.0" \
    WEBSOCKIFY_VERSION="0.13.0" \
    OPT_PATH="/opt"
RUN  wget -nv -O noVNC.zip "https://github.com/novnc/noVNC/archive/refs/tags/v${NOVNC_VERSION}.zip" \
 && unzip -x noVNC.zip \
 && rm noVNC.zip  \
 && mv noVNC-${NOVNC_VERSION} ${OPT_PATH}/noVNC \
 && wget -nv -O websockify.zip "https://github.com/novnc/websockify/archive/refs/tags/v${WEBSOCKIFY_VERSION}.zip" \
 && unzip -x websockify.zip \
 && mv websockify-${WEBSOCKIFY_VERSION} ${OPT_PATH}/noVNC/utils/websockify \
 && rm websockify.zip \
 && ln ${OPT_PATH}/noVNC/vnc.html ${OPT_PATH}/noVNC/index.html

ENV DISPLAY=:0 \
    SCREEN_NUMBER=0 \
    SCREEN_WIDTH=1600 \
    SCREEN_HEIGHT=900 \
    SCREEN_DEPTH=24+32 \
    VNC_PORT=5900 \
    WEB_VNC_PORT=6080

EXPOSE 5900 6080

#==========
# Copy app
#==========
RUN rm -rf ${SCRIPT_PATH}
ENV SCRIPT_PATH="docker-android"
ENV APP_PATH=${WORK_PATH}/${SCRIPT_PATH}
RUN mkdir -p ${APP_PATH}
COPY mixins ${APP_PATH}/mixins
COPY cli ${APP_PATH}/cli
# NepStack fork: upstream's private build-secret step replaced by the open-source CLI,
# exposed as /usr/local/bin/docker-android (used by the supervisord configs).
RUN PIP_BREAK_SYSTEM_PACKAGES=1 pip install --no-cache-dir click requests \
 && printf '#!/bin/bash\nexport PYTHONPATH="%s/cli/src"\nexec python3 "%s/cli/src/app.py" "$@"\n' "${APP_PATH}" "${APP_PATH}" > /usr/local/bin/docker-android \
 && chmod 755 /usr/local/bin/docker-android \
 && chmod +x ${APP_PATH}/mixins/scripts/*.sh \
 && chown -R ${USERID}:${GROUPID} ${APP_PATH}
RUN echo "${USERNAME} ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers

#===================
# Configure OpenBox
#===================
RUN mkdir -p /etc/xdg/openbox \
 && echo ${APP_PATH}/mixins/configs/display/.fehbg >> /etc/xdg/openbox/autostart

#==================
# Use created user
#==================
USER ${USERID}:${GROUPID}
ENV LOG_PATH=${WORK_PATH}/logs \
    WEB_LOG_PORT=9000
EXPOSE 9000
RUN mkdir -p ${LOG_PATH}
RUN mkdir -p "${WORK_PATH}/.config/Android Open Source Project" \
 && echo "[General]\nshowNestedWarning=false\n" > "${WORK_PATH}/.config/Android Open Source Project/Emulator.conf"

#=========
# Run App
#=========
STOPSIGNAL SIGTERM
ENV DEVICE_TYPE=emulator
CMD ["${APP_PATH}/mixins/scripts/run.sh"]
