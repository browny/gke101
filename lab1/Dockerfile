# A docker base image based on alpine linux with python 3.6.0 pre-installed.
# Docker images can be stacked on top of each other.
FROM library/python:3.6.0-alpine

# Install tornado library.
RUN pip install tornado

# Copy the webserver python app.
ADD web-server.py /web-server.py

# Set the default execution command for this image.
CMD ["python", "/web-server.py"]
