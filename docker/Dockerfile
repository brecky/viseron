ARG ARCH
ARG BASE_VERSION
ARG OPENCV_VERSION
ARG FFMPEG_VERSION
ARG WHEELS_VERSION
ARG GPAC_IMAGE
ARG UBUNTU_VERSION
ARG NODE_VERSION
FROM brecky/${ARCH}-opencv:${OPENCV_VERSION} AS opencv
FROM brecky/${ARCH}-ffmpeg:${FFMPEG_VERSION} AS ffmpeg
FROM brecky/${ARCH}-wheels:${WHEELS_VERSION} AS wheels
FROM ${GPAC_IMAGE} AS gpac

# Install go2rtc
FROM brecky/${ARCH}-base:${BASE_VERSION} AS go2rtc-base
ARG ARCH
ARG GO2RTC_VERSION

WORKDIR /usr/local/bin

RUN \
  apt-get update && apt-get install -y --no-install-recommends \
  wget

RUN if [ "$ARCH" = "amd64-cuda" ]; then PLATFORM="amd64"; \
  elif [ "$ARCH" = "armhf" ] || [ "$ARCH" = "rpi3" ]; then PLATFORM="arm"; \
  elif [ "$ARCH" = "aarch64" ] || [ "$ARCH" = "jetson-nano" ]; then PLATFORM="arm64"; \
  else PLATFORM="$ARCH"; fi \
  && wget https://github.com/AlexxIT/go2rtc/releases/download/v${GO2RTC_VERSION}/go2rtc_linux_$PLATFORM -O go2rtc \
  && chmod +x /usr/local/bin/go2rtc

WORKDIR /go2rtc
RUN wget https://raw.githubusercontent.com/AlexxIT/go2rtc/refs/tags/v${GO2RTC_VERSION}/www/video-rtc.js -O video-rtc.js

FROM scratch AS go2rtc
COPY --from=go2rtc-base /usr/local/bin/go2rtc /usr/local/bin/go2rtc
COPY --from=go2rtc-base /go2rtc/video-rtc.js /go2rtc/


# Build frontend
FROM node:${NODE_VERSION} AS frontend

ARG NODE_OPTIONS=""
ENV NODE_OPTIONS=${NODE_OPTIONS}

WORKDIR /frontend

COPY frontend/package.json ./
COPY frontend/package-lock.json ./

RUN npm ci --legacy-peer-deps

COPY frontend/ ./
COPY --from=go2rtc /go2rtc/video-rtc.js /frontend/src/components/player/liveplayer/video-rtc.js
RUN npm run build


# Build Viseron
FROM brecky/${ARCH}-base:${BASE_VERSION}

WORKDIR /src

COPY --from=wheels /wheels /wheels
COPY --from=wheels /wheels-3.9 /wheels-3.9

ARG S6_OVERLAY_ARCH
ARG S6_OVERLAY_VERSION
ARG EXTRA_APT_PACKAGES

ENV \
  VISERON_CONFIG_DIR=/config \
  DEBIAN_FRONTEND=noninteractive \
  S6_KEEP_ENV=1 \
  S6_SERVICES_GRACETIME=30000 \
  S6_KILL_GRACETIME=30000 \
  S6_KILL_FINISH_MAXTIME=30000 \
  PATH=$PATH:/home/abc/bin \
  LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/lib:/ffmpeg/lib \
  PYTHONPATH=$PYTHONPATH:/usr/local/lib/python3.10/site-packages \
  OPENCV_OPENCL_CACHE_ENABLE=false \
  PG_COLOR="always" \
  PYTHONUNBUFFERED=1

ADD https://github.com/just-containers/s6-overlay/releases/download/v${S6_OVERLAY_VERSION}/s6-overlay-${S6_OVERLAY_ARCH}-installer /tmp/s6-overlay-installer

RUN \
  apt-get update \
  && apt-get install -y --no-install-recommends \
  curl \
  cpulimit \
  gnupg \
  tzdata \
  python3 \
  python3-pip \
  python3-dev \
  python3-setuptools \
  python3-wheel \
  libgirepository1.0-dev\
  usbutils \
  # OpenCV and FFmpeg Dependencies
  libgomp1 \ 
  # dlib Dependencies openblas 
  libopenblas-dev \
  # PostgreSQL
  postgresql \
  postgresql-contrib \
  #NGNX 
  nginx \ 
  \
  # Install extra apt packages from build-arg
  ${EXTRA_APT_PACKAGES} \
  # GStreamer
  libgstreamer1.0-dev \
  libgstreamer-plugins-base1.0-dev \
  libgstreamer-plugins-bad1.0-dev \
  gstreamer1.0-alsa \
  gstreamer1.0-libav \
  gstreamer1.0-plugins-base \
  gstreamer1.0-plugins-good \
  gstreamer1.0-plugins-bad \
  gstreamer1.0-plugins-ugly \
  gstreamer1.0-tools \
  gstreamer1.0-vaapi \
  gstreamer1.0-gl \
  gstreamer1.0-pulseaudio \
  python3-gst-1.0 \ 
  software-properties-common \
  #&& rm -rf /var/lib/apt/lists/* \
  && apt-get purge pip \
  python3-pyparsing \
  python3-packaging \
  python3-numpy \
  python3-gi -y 
# Install Python 3.9 if it doesn't exist
  RUN if [ ! -f /usr/local/bin/python3.11 ]; then \
  add-apt-repository ppa:deadsnakes/ppa -y && \
  apt-get update && apt-get install --no-install-recommends -y \
  python3.11 \
  python3.11-distutils \
  && curl -s https://bootstrap.pypa.io/get-pip.py | python3.11 - ; \
  fi 

  RUN python3 -m pip install pip --break-system-packages 
  RUN python3 -m pip install /wheels/*.whl --break-system-packages  
  RUN rm -r /wheels 
  
 
   # Install Coral Edge TPU
  RUN echo "deb https://packages.cloud.google.com/apt coral-edgetpu-stable main" | tee /etc/apt/sources.list.d/coral-edgetpu.list \
  && curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add - \
  && echo "libedgetpu1-max libedgetpu/accepted-eula select true" | debconf-set-selections \
  && apt-get -qq update && apt-get -qq install -y --no-install-recommends \
  libedgetpu1-max 
  RUN python3.11 -m pip install /wheels-3.9/*.whl --break-system-packages\
  && rm -r /wheels-3.11 \
  \
  && chmod +x /tmp/s6-overlay-installer && /tmp/s6-overlay-installer / \
  && apt-get autoremove -y \
  && apt-get clean -y \
  && rm -rf /var/lib/apt/lists/* \
  && rm -rf /wheels \
  && rm -r /tmp/s6-overlay-installer \
  && useradd --uid 911 --user-group --create-home abc \
  && mkdir -p /home/abc/bin /segments

ENTRYPOINT ["/init"]

COPY docker/ffprobe_wrapper /home/abc/bin/ffprobe
COPY docker/ffmpeg_wrapper /home/abc/bin/ffmpeg
COPY --from=opencv /opt/opencv /usr/local/
COPY --from=ffmpeg /usr/local /ffmpeg/
COPY --from=gpac /gpac/bin/gcc /usr/bin
COPY --from=go2rtc /usr/local/bin/go2rtc /usr/local/bin/go2rtc
COPY rootfs/ /
COPY --from=frontend /frontend/dist /src/viseron/frontend/
COPY manager.py /src/manager.py
COPY viseron /src/viseron/

ARG VISERON_VERSION="dev"
ARG VISERON_GIT_COMMIT="unknown"
ENV VISERON_VERSION=${VISERON_VERSION}
ENV VISERON_GIT_COMMIT=${VISERON_GIT_COMMIT}