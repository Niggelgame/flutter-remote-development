FROM codercom/code-server:latest

# Install Flutter
RUN sudo apt-get update
RUN sudo apt-get install -y unzip
# Usually in a normal Linux environment, you should not do that
RUN sudo chmod 777 /opt
RUN git clone https://github.com/flutter/flutter.git -b dev --depth 1 /opt/flutter
ENV PATH="/opt/flutter/bin:${PATH}"
RUN flutter doctor
RUN flutter config --enable-web

# Install Code extensions
RUN code-server --install-extension dart-code.flutter
