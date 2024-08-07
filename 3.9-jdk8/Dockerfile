##
# osgeo/gdal:alpine-small

# This file is available at the option of the licensee under:
# Public domain
# or licensed under MIT (LICENSE.TXT) Copyright 2019 Even Rouault <even.rouault@spatialys.com>

ARG ALPINE_VERSION=3.19
FROM amazoncorretto:8u412-alpine${ALPINE_VERSION} AS builder-jdk

# Setup build env for PROJ
RUN apk add --no-cache wget curl unzip make libtool autoconf automake pkgconfig g++ sqlite sqlite-dev

# For PROJ and GDAL
RUN apk add --no-cache \
    linux-headers \
    curl-dev tiff-dev \
    zlib-dev zstd-dev lz4-dev libdeflate-dev libarchive-dev \
    libjpeg-turbo-dev libpng-dev libwebp-dev expat-dev postgresql-dev openjpeg-dev

RUN apk add --no-cache cmake rsync ccache swig


RUN wget -O apache-ant-1.10.14-bin.tar.gz https://downloads.apache.org/ant/binaries/apache-ant-1.10.14-bin.tar.gz && \
    tar x -ovzf apache-ant-1.10.14-bin.tar.gz && \
    mv apache-ant-1.10.14 /usr/share/ant && \
    ln -s /usr/share/ant/bin/ant /usr/bin/ant && \
    rm -f apache-ant-1.10.14-bin.tar.gz


FROM builder-jdk AS builder-gdal

# Build openjpeg
#ARG OPENJPEG_VERSION=2.3.1
RUN if test "${OPENJPEG_VERSION}" != ""; then ( \
    wget -q https://github.com/uclouvain/openjpeg/archive/v${OPENJPEG_VERSION}.tar.gz \
    && tar xzf v${OPENJPEG_VERSION}.tar.gz \
    && rm -f v${OPENJPEG_VERSION}.tar.gz \
    && cd openjpeg-${OPENJPEG_VERSION} \
    && cmake . -DBUILD_SHARED_LIBS=ON  -DBUILD_STATIC_LIBS=OFF -DCMAKE_BUILD_TYPE=Release \
        -DCMAKE_INSTALL_PREFIX=/usr \
    && make -j$(nproc) \
    && make install \
    && mkdir -p /build_thirdparty/usr/lib \
    && cp -P /usr/lib/libopenjp2*.so* /build_thirdparty/usr/lib \
    && for i in /build_thirdparty/usr/lib/*; do strip -s $i 2>/dev/null || /bin/true; done \
    && cd .. \
    && rm -rf openjpeg-${OPENJPEG_VERSION} \
    ); fi


ARG PROJ_DATUMGRID_LATEST_LAST_MODIFIED
RUN \
    mkdir -p /build_projgrids/usr/share/proj \
    && curl -LOs http://download.osgeo.org/proj/proj-datumgrid-latest.zip \
    && unzip -q -j -u -o proj-datumgrid-latest.zip  -d /build_projgrids/usr/share/proj \
    && rm -f *.zip

ARG RSYNC_REMOTE

# Build PROJ
ARG PROJ_VERSION=master
RUN mkdir proj \
    && wget -q https://github.com/OSGeo/PROJ/archive/${PROJ_VERSION}.tar.gz -O - \
        | tar xz -C proj --strip-components=1 \
    && cd proj \
    && if test "${RSYNC_REMOTE}" != ""; then \
        echo "Downloading cache..."; \
        rsync -ra ${RSYNC_REMOTE}/proj/$(uname -m)/ $HOME/; \
        echo "Finished"; \
        export CC="ccache gcc"; \
        export CXX="ccache g++"; \
        mkdir -p "$HOME/.ccache"; \
        export PROJ_DB_CACHE_DIR="$HOME/.ccache"; \
        ccache -M 100M; \
    fi \
    # IPO disabled since it crashes with gcc 13.2.1
    && cmake . \
        -DBUILD_SHARED_LIBS=ON \
        -DCMAKE_BUILD_TYPE=Release \
        -DCMAKE_INSTALL_PREFIX=/usr \
        -DENABLE_IPO=OFF \
        -DBUILD_TESTING=OFF \
    && make -j$(nproc) \
    && make install \
    && make install DESTDIR="/build_proj" \
    && if test "${RSYNC_REMOTE}" != ""; then \
        ccache -s; \
        echo "Uploading cache..."; \
        rsync -ra --delete $HOME/.ccache ${RSYNC_REMOTE}/proj/$(uname -m)/; \
        echo "Finished"; \
        rm -rf $HOME/.ccache; \
        unset CC; \
        unset CXX; \
    fi \
    && cd .. \
    && rm -rf proj \
    && for i in /build_proj/usr/lib/*; do strip -s $i 2>/dev/null || /bin/true; done \
    && for i in /build_proj/usr/bin/*; do strip -s $i 2>/dev/null || /bin/true; done 

# Build GDAL
ARG GDAL_VERSION=3.9.0
ARG GDAL_RELEASE_DATE
ARG GDAL_BUILD_IS_RELEASE
ARG GDAL_REPOSITORY=OSGeo/gdal

RUN if test "${RSYNC_REMOTE}" != ""; then \
        echo "Downloading cache..."; \
        rsync -ra ${RSYNC_REMOTE}/gdal/$(uname -m)/ $HOME/; \
        echo "Finished"; \
        export CC="ccache gcc"; \
        export CXX="ccache g++"; \
        mkdir -p "$HOME/.ccache"; \
        ccache -M 1G; \
    fi \
    && mkdir gdal \
    && wget -q https://github.com/${GDAL_REPOSITORY}/releases/download/v${GDAL_VERSION}/gdal-${GDAL_VERSION}.tar.gz -O - \
        | tar xz -C gdal --strip-components=1 \
    && cd gdal \
    && mkdir build \
    && cd build \
    && cmake .. \
        -DCMAKE_INSTALL_PREFIX=/usr \
        -DCMAKE_BUILD_TYPE=Release \
        -DGDAL_USE_TIFF_INTERNAL=ON \
        -DGDAL_USE_GEOTIFF_INTERNAL=ON \
        -DBUILD_TESTING=OFF \
        -DBUILD_JAVA_BINDINGS:BOOL=ON \
    && make -j$(nproc) \
    && make install DESTDIR="/build" \
    && cd .. \
    && if test "${RSYNC_REMOTE}" != ""; then \
        ccache -s; \
        echo "Uploading cache..."; \
        rsync -ra --delete $HOME/.ccache ${RSYNC_REMOTE}/gdal/$(uname -m)/; \
        echo "Finished"; \
        rm -rf $HOME/.ccache; \
        unset CC; \
        unset CXX; \
    fi \
    && cd .. \
    && rm -rf gdal \
    && mkdir -p /build_gdal_version_changing/usr/include \
    && mv /build/usr/lib                    /build_gdal_version_changing/usr \
    && mv /build/usr/include/gdal_version.h /build_gdal_version_changing/usr/include \
    && mv /build/usr/bin                    /build_gdal_version_changing/usr \
    && for i in /build_gdal_version_changing/usr/lib/*; do strip -s $i 2>/dev/null || /bin/true; done \
    && for i in /build_gdal_version_changing/usr/bin/*; do strip -s $i 2>/dev/null || /bin/true; done 

# Build final image
FROM amazoncorretto:8u412-alpine${ALPINE_VERSION}-jre AS runner

LABEL maintainer="Sean Wan <haies@live.com>"

RUN date
RUN apk add --no-cache \
        libstdc++ \
        sqlite-libs \
        libcurl tiff \
        zlib zstd-libs lz4-libs libdeflate libarchive \
        libjpeg-turbo libpng openjpeg libwebp expat libpq \
        freetype fontconfig ttf-dejavu \
    # libturbojpeg.so is not used by GDAL. Only libjpeg.so*
    && rm -f /usr/lib/libturbojpeg.so* \
    # Only libwebp.so is used by GDAL
    && rm -f /usr/lib/libwebpmux.so* /usr/lib/libwebpdemux.so* /usr/lib/libwebpdecoder.so*

# Order layers starting with less frequently varying ones
#COPY --from=builder  /build_thirdparty/usr/ /usr/

COPY ./fonts/ /usr/share/fonts/
RUN fc-cache -fv
COPY --from=builder-gdal  /build_projgrids/usr/ /usr/

COPY --from=builder-gdal  /build_proj/usr/share/proj/ /usr/share/proj/
COPY --from=builder-gdal  /build_proj/usr/include/ /usr/include/
COPY --from=builder-gdal  /build_proj/usr/bin/ /usr/bin/
COPY --from=builder-gdal  /build_proj/usr/lib/ /usr/lib/

COPY --from=builder-gdal  /build/usr/share/gdal/ /usr/share/gdal/
COPY --from=builder-gdal  /build/usr/include/ /usr/include/
COPY --from=builder-gdal  /build_gdal_version_changing/usr/ /usr/

ENTRYPOINT ["/bin/sh", "-c","for jar in /jars/*.jar; do java $JAVA_OPTS -jar $jar; done"]
